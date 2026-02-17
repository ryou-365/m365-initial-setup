# 03_TenantCoreSettings

* テナント全体に影響する、不可逆・横断的な基本設定（組織情報・通知・既定値方針）

## ねらい

本章では、後続のセキュリティ強化（04_ConditionalAccessBaseline / 05_ServiceSecurityBaseline）や運用整備（06以降）に入る前に、
**テナントとしての“名刺”と“連絡先”と“既定値”**を揃えます。

* **組織情報**（Organization profile / Privacy profile）を整備し、外部向けの説明責任を持たせる
* **通知先**（技術・セキュリティ/コンプライアンス）を定義し、運用の入口を作る
* **既定値方針**（SharePoint/OneDrive の共有リンク既定）を定め、現場が迷わない初期値を作る

> 注意:
>
> * 本章は「監査ログの有効化」ではありません（02で完了済み）。
> * 条件付きアクセスやMFA強制などの“防御”は 04。
> * Exchange の“強い”セキュリティ既定（認証経路/通信経路など）は 05。

---

## 位置づけ（README との対応）

* 区分: **Mandatory（後から直せない・全体に効く）**
* 対象: `03_TenantCoreSettings`
* 前提:

  * `00_Initialization` が完了していること
  * `01_EmergencyAccess` が完了していること
  * `02_AuditAndBaseline` が完了していること

---

# Phase 0: Tenant Variables（EDIT HERE ONLY）

> 00_Initialization の Phase 0 を参照してください（同一内容を再掲しません）。

- [00_Initialization.md #phase-0-initial-setup-parameter-sheet-v10--tenant-variables](https://github.com/ryou-365/m365-initial-setup/blob/main/00_Initialization.md#phase-0-initial-setup-parameter-sheet-v10--tenant-variables)

---

# Phase 1: 前提確認・接続（Graph / EXO / SPO）

> 注意:
>
> * 00_Initialization で定めた通り、Graph/EXO/SPO は **別々のPowerShellセッション**で実行してください。
> * 本章は設定変更を含むため、接続後に必ずテナント識別（表示名/URL）を目視で確認します。

---

## 1.1 Microsoft Graph（Organization 情報更新）

> 必要スコープ: `Organization.ReadWrite.All`

```powershell
Disconnect-MgGraph -ErrorAction SilentlyContinue

Connect-MgGraph `
  -TenantId $TenantFallbackDomain `
  -Scopes "Organization.ReadWrite.All" `
  -UseDeviceCode `
  -ContextScope Process

(Get-MgContext).Account
(Get-MgContext).TenantId
```

---

## 1.2 Exchange Online（通知/DSN既定）

```powershell
Disconnect-ExchangeOnline -Confirm:$false -ErrorAction SilentlyContinue
Connect-ExchangeOnline -UserPrincipalName $AdminUpn -ShowBanner:$false -ErrorAction SilentlyContinue

Get-OrganizationConfig | Select-Object DisplayName
```

---

## 1.3 SharePoint Online（共有リンク既定）

```powershell
Import-Module Microsoft.Online.SharePoint.PowerShell -Force

Disconnect-SPOService -ErrorAction SilentlyContinue
Connect-SPOService -Url $SPOAdminUrl -UseSystemBrowser:$true

(Get-SPOSite -Identity $SPORootUrl -ErrorAction Stop) | Select-Object Url, Title
```

---

# Phase 2: Organization Profile（Graph）

## 2.1 現状確認（目視）

```powershell
$org = Get-MgOrganization | Select-Object -First 1
$org | Select-Object Id, DisplayName, PreferredLanguage, MarketingNotificationEmails, TechnicalNotificationMails, SecurityComplianceNotificationMails

# PrivacyProfile は入れ子なので個別に確認
$org.PrivacyProfile
```

---

## 2.2 更新（この章では実施しない）

> 方針:
>
> * Organization の通知先は、Phase 3 で共有メールボックス作成＋外部転送を設定した後に **Phase 3.4** で集約します。
> * ここ（Phase 2）では **現状確認のみ** とします。

---

# Phase 3: Exchange Online（共有メールボックス / 通知経路）

> 目的:
>
> * 通知を「テナント内に集約」する
> * かつ「外部独立経路」へ自動転送する
> * アクセス権は内部ユーザーのみに限定する

## 3.1 共有メールボックス作成（冪等）

```powershell
function Ensure-SharedMailbox {
  param(
    [string]$DisplayName,
    [string]$Alias,
    [string]$PrimarySmtpAddress
  )

  $mb = Get-Mailbox -Identity $PrimarySmtpAddress -ErrorAction SilentlyContinue
  if (-not $mb) {
    New-Mailbox -Shared -Name $DisplayName -Alias $Alias -PrimarySmtpAddress $PrimarySmtpAddress
  }
}

Ensure-SharedMailbox -DisplayName $SharedMailbox_TechOps_DisplayName -Alias $SharedMailbox_TechOps_Alias -PrimarySmtpAddress $SharedMailbox_TechOps_Address
Ensure-SharedMailbox -DisplayName $SharedMailbox_SecOps_DisplayName  -Alias $SharedMailbox_SecOps_Alias  -PrimarySmtpAddress $SharedMailbox_SecOps_Address
Ensure-SharedMailbox -DisplayName $SharedMailbox_Privacy_DisplayName -Alias $SharedMailbox_Privacy_Alias -PrimarySmtpAddress $SharedMailbox_Privacy_Address
```

---

## 3.2 内部メンバー付与（冪等）

```powershell
function Ensure-MailboxMembers {
  param(
    [string]$Mailbox,
    [string[]]$Members
  )

  foreach ($m in $Members) {
    $perm = Get-MailboxPermission -Identity $Mailbox -User $m -ErrorAction SilentlyContinue
    if (-not $perm) {
      Add-MailboxPermission -Identity $Mailbox -User $m -AccessRights FullAccess -InheritanceType All
    }
  }
}

Ensure-MailboxMembers -Mailbox $SharedMailbox_TechOps_Address -Members $SharedMailbox_TechOps_InternalMembers
Ensure-MailboxMembers -Mailbox $SharedMailbox_SecOps_Address  -Members $SharedMailbox_SecOps_InternalMembers
Ensure-MailboxMembers -Mailbox $SharedMailbox_Privacy_Address -Members $SharedMailbox_Privacy_InternalMembers
```

---

## 3.3 外部転送設定（冪等）

```powershell
function Ensure-ExternalForwarding {
  param(
    [string]$Mailbox,
    [string[]]$Recipients
  )

  foreach ($r in $Recipients) {
    $contact = Get-MailContact -Identity $r -ErrorAction SilentlyContinue
    if (-not $contact) {
      $alias = ($r -replace "@","-")
      New-MailContact -Name $alias -ExternalEmailAddress $r
    }
  }

  Set-Mailbox -Identity $Mailbox -ForwardingSmtpAddress $Recipients[0] -DeliverToMailboxAndForward $true
}

Ensure-ExternalForwarding -Mailbox $SharedMailbox_TechOps_Address -Recipients $SharedMailbox_TechOps_ExternalForwarding
Ensure-ExternalForwarding -Mailbox $SharedMailbox_SecOps_Address  -Recipients $SharedMailbox_SecOps_ExternalForwarding
Ensure-ExternalForwarding -Mailbox $SharedMailbox_Privacy_Address -Recipients $SharedMailbox_Privacy_ExternalForwarding
```

---

## 3.4 Organization 通知宛先を共有メールボックスへ設定

> 共有メールボックス作成（3.1）と外部転送（3.3）を終えた後に、Microsoft 側の通知先をここへ集約します。

```powershell
$org = Get-MgOrganization | Select-Object -First 1

$params = @{
  preferredLanguage = $OrgPreferredLanguage

  # 通知先は共有メールボックスに集約
  technicalNotificationMails          = @($SharedMailbox_TechOps_Address)
  securityComplianceNotificationMails = @($SharedMailbox_SecOps_Address)

  # Privacy は外部提示情報（メールは共有メールボックス、URLは公開URL）
  privacyProfile = @{
    contactEmail = $OrgPrivacyContactEmail
    statementUrl = $OrgPrivacyStatementUrl
  }
}

Update-MgOrganization -OrganizationId $org.Id -BodyParameter $params
```

---

powershell
$org = Get-MgOrganization | Select-Object -First 1

$params = @{
preferredLanguage = $OrgPreferredLanguage
technicalNotificationMails = @($SharedMailbox_TechOps_Address)
securityComplianceNotificationMails = @($SharedMailbox_SecOps_Address)
privacyProfile = @{
contactEmail = $SharedMailbox_Privacy_Address
}
}

Update-MgOrganization -OrganizationId $org.Id -BodyParameter $params

````

---

# Phase 4: SharePoint/OneDrive（共有リンク既定値）（共有リンク既定値）

> 目的:
>
> * “デフォルトで事故りにくい” を先に作る
> * 後続（07）でサイト設計やストレージ設計に進む前に、まず全体既定だけ揃える

## 4.1 現状確認

```powershell
Get-SPOTenant | Select-Object SharingCapability, DefaultSharingLinkType, DefaultLinkPermission, RequireAcceptingAccountMatchInvitedAccount, BccExternalSharingInvitations
````

---

## 4.2 共有リンク既定を設定（冪等）

> 推奨（例）:
>
> * DefaultSharingLinkType = Direct（既存アクセス/特定ユーザー共有をデフォルトに寄せる）
> * DefaultLinkPermission  = View（編集リンクをデフォルトにしない）
> * RequireAcceptingAccountMatchInvitedAccount = True（招待先と受諾アカウント一致要求）
>
> ※ 組織の方針により `SharingCapability` などは 07 で扱うのが自然な場合があります。
> 本章では「リンクの既定値」中心に留めます。

```powershell
$tenant = Get-SPOTenant

# DefaultSharingLinkType
if ($tenant.DefaultSharingLinkType.ToString() -ne $SPO_DefaultSharingLinkType) {
  Set-SPOTenant -DefaultSharingLinkType $SPO_DefaultSharingLinkType
}

# DefaultLinkPermission
if ($tenant.DefaultLinkPermission.ToString() -ne $SPO_DefaultLinkPermission) {
  Set-SPOTenant -DefaultLinkPermission $SPO_DefaultLinkPermission
}

# 招待の受諾アカウント一致（外部共有の基本ブレーキ）
if ($tenant.RequireAcceptingAccountMatchInvitedAccount -ne $true) {
  Set-SPOTenant -RequireAcceptingAccountMatchInvitedAccount $true
}

# 外部共有招待のBCC（窓口で追跡したい場合のみ：必要なら有効化）
# if ($tenant.BccExternalSharingInvitations -ne $true) {
#   Set-SPOTenant -BccExternalSharingInvitations $true -BccExternalSharingInvitationsList "it-ops@example.com"
# }

# 再確認
Get-SPOTenant | Select-Object SharingCapability, DefaultSharingLinkType, DefaultLinkPermission, RequireAcceptingAccountMatchInvitedAccount, BccExternalSharingInvitations
```

---

# Phase 5: 切断

```powershell
Disconnect-MgGraph -ErrorAction SilentlyContinue
Disconnect-ExchangeOnline -Confirm:$false -ErrorAction SilentlyContinue
Disconnect-SPOService -ErrorAction SilentlyContinue
```

---

# 完了条件（チェック）

* [ ] Organization の TechnicalNotificationMails / SecurityComplianceNotificationMails が意図した値になっている
* [ ] PrivacyProfile（contactEmail / statementUrl）が設定されている
* [ ] Exchange の ExternalPostmasterAddress が意図した宛先になっている
* [ ] SharePoint の DefaultSharingLinkType / DefaultLinkPermission が意図した既定になっている

---

# 補足（設計メモ）

* 03は「全体に効く初期値」を固める章。
* “強い制御”は 04/05 に寄せ、ここでは **運用の入口と既定値**を先に作る。
* 既定値を先に作ることで、後続の設計（07_CollaborationAndStorage）やリリース（社内展開）での説明コストが下がる。
