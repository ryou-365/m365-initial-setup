# 00_Initialization

- 実行環境セットアップ & 接続テスト（設定変更なし）

## ねらい

* **A：環境整備**（PS7 / モジュール導入・保守）を行う

  * 基本は「最初の 1 回」＋「保守（必要時）」
* **B：接続テスト**（Graph / EXO / SPO / Teams）を行う

  * 01 以降で繰り返し発生する接続操作を、**シンプルな型**として確立する
* 本章は **設定変更を行わない**（接続・参照のみ）

---

# A：環境整備（端末側の準備）

## A-1. PowerShell 7 のインストール

```powershell
winget install --id Microsoft.Powershell --source winget
```

## A-2. PS7 を管理者で起動し直す（目視）

* いったん PowerShell ウィンドウをすべて閉じる
* **スタートメニューのアプリ一覧か「PowerShell 7 (x64)」** を「管理者として実行」で起動する

## A-3. PS7 起動確認（目視）

```powershell
$PSVersionTable.PSVersion
```

---

## A-4. モジュール導入方針（重要）

* `Install-Module` は **必ず `-Scope AllUsers`**

  * `CurrentUser` は OneDrive の Documents 配下に入り得て、同期・競合の影響を受ける可能性があるため
* SharePoint Online 管理モジュール（`Microsoft.Online.SharePoint.PowerShell`）は **WinPS(5.1) 互換**が必要
* PnP.PowerShell は使用しない（**純正モジュールのみ**）

---

## A-5. モジュールの導入（初回）

### Microsoft Graph

```powershell
Install-Module Microsoft.Graph -Repository PSGallery -Scope AllUsers -Force
```

### Exchange Online

```powershell
Install-Module ExchangeOnlineManagement -Repository PSGallery -Scope AllUsers -Force
```

### Microsoft Teams

```powershell
Install-Module MicrosoftTeams -Repository PSGallery -Scope AllUsers -Force
```

### SharePoint Online（導入は WinPS 5.1 側で行う）

> **Windows PowerShell 5.1（管理者）**で実行してください。

```powershell
$PSVersionTable.PSVersion
```

```powershell
Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force
```

```powershell
Install-Module Microsoft.Online.SharePoint.PowerShell -Repository PSGallery -Scope AllUsers -Force
```

---

## A-6. モジュールの保守（任意）

### 導入済みバージョンの確認

```powershell
Get-InstalledModule Microsoft.Graph,ExchangeOnlineManagement,MicrosoftTeams -ErrorAction SilentlyContinue | Select-Object Name,Version,InstalledLocation
Get-Module -ListAvailable Microsoft.Online.SharePoint.PowerShell | Sort-Object Version -Descending | Select-Object -First 1 Name,Version,Path
```

### 更新（必要時のみ実行）

```powershell
Update-Module Microsoft.Graph -Scope AllUsers -Force
```

```powershell
Update-Module ExchangeOnlineManagement -Scope AllUsers -Force
```

```powershell
Update-Module MicrosoftTeams -Scope AllUsers -Force
```

```powershell
Update-Module Microsoft.Online.SharePoint.PowerShell -Scope AllUsers -Force
```

---

# B：接続テスト（使い回し前提・インタラクティブ認証）

> 要件:
>
> * 01 以降で **そのまま流用**できる
> * トラブルが少なく **シンプル**
> * 基本は **インタラクティブ認証**（ブラウザ認証）
> * モジュールのインストール／更新は **含めない**（A で実施）

> 注意:
>
> * Graph/Exchange Online/Teams/SharePoint Onlineへの接続は、それぞれ別でPowerShellを起動して実施してください。
> * 原則は、常に1つだけで作業し、同時作業は並行しないようにしてください。
> * 接続の際に利用するMSALが混在すると認証時に高い確率でトラブルが発生します。


## Phase 0: Initial Setup Parameter Sheet v1.0  (Tenant Variables)

- 下記をコピーして手元で自身の環境に応じた設定を記述し、接続前にPowerShellで実行してください。

```powershell
<# =====================================================================
Initial Setup Parameter Sheet v1.0  (Tenant Variables)

目的:
- 初期構築の「設計決定事項」を1箇所に集約し、再現性と説明責任を確保する。
- EDIT HERE ONLY 以外を編集しないことで、派生値の整合性を崩さない。

運用ルール:
- 編集してよいのは「EDIT HERE ONLY」だけ。
- 変更したら、このファイルの末尾 Change Log に理由と日付を残す（任意だが推奨）。
===================================================================== #>

# ==========================================================
# EDIT HERE ONLY: 00_Initialization: Tenant Identity
# ==========================================================
# [設計意図]
# - TenantCustomDomain は社内利用の主ドメイン。
# - TenantOnMicrosoft はテナント作成時に決まる固定値（後から変更不可）。

$TenantCustomDomain = "example.com"
$TenantOnMicrosoft  = "example-fallback"

# [設計意図]
# - AdminUser は初期構築の作業主体アカウント。
# - break-glass は 01_EmergencyAccess で作成する「最後の入口」なので、命名は固定化する。

$AdminUser = "admin"

$BreakGlassUser1 = "breakglass1"
$BreakGlassUser2 = "breakglass2"

$BreakGlassDisplayName1 = "BreakGlass Account 1"
$BreakGlassDisplayName2 = "BreakGlass Account 2"

# [設計意図]
# - CA除外は「グループ1つ」に集約し、レビュー対象をルール変更のみに限定する。
$BreakGlassExemptGroupName     = "BreakGlass Exempt (CA Exclusion)"
$BreakGlassExemptGroupMailNick = "sg-entra-breakglass-exempt"


# ==========================================================
# EDIT HERE ONLY: 03_TenantCoreSettings: Notification Mailboxes
# ==========================================================
# [設計方針]
# - 通知先は「共有メールボックス + 外部自動転送」で担保する。
# - 外部アドレスは“アクセス権”ではなく“転送先”で扱う（外部ユーザーに権限は付与しない）。
# - 各共有メールボックスの内部メンバー / 外部転送先は “それぞれ異なる前提”。

# 技術運用通知用
$SharedMailbox_TechOps_DisplayName = "IT Operations"
$SharedMailbox_TechOps_Alias       = "it-ops"
$SharedMailbox_TechOps_InternalMembers = @(
  # "user1@example.com"
)
$SharedMailbox_TechOps_ExternalForwarding = @(
  # "external-tech@example-external.net"
)

# セキュリティ通知用
$SharedMailbox_SecOps_DisplayName = "Security Operations"
$SharedMailbox_SecOps_Alias       = "sec-ops"
$SharedMailbox_SecOps_InternalMembers = @(
  # "user2@example.com"
)
$SharedMailbox_SecOps_ExternalForwarding = @(
  # "external-sec@example-external.net"
)

# プライバシー窓口用
$SharedMailbox_Privacy_DisplayName = "Privacy Contact"
$SharedMailbox_Privacy_Alias       = "privacy"
$SharedMailbox_Privacy_InternalMembers = @(
  # "user3@example.com"
)
$SharedMailbox_Privacy_ExternalForwarding = @(
  # "external-privacy@example-external.net"
)


# ==========================================================
# EDIT HERE ONLY: 03_TenantCoreSettings: Organization Profile / SPO Defaults
# ==========================================================
# [設計意図]
# - preferredLanguage: 管理・通知メッセージの既定言語（運用チームの言語に揃える）
$OrgPreferredLanguage = "ja-JP"

# [設計意図]
# - Privacy は外部提示情報。メールは privacy@ に統一し、URLは公開ポリシーを指定する。
$OrgPrivacyStatementUrl = "https://example.com/privacy"

# [選定基準: SharePoint/OneDrive 共有リンク既定]
# - DefaultSharingLinkType:
#   Direct  : 共有先を明示（リンク流出事故を起こしにくい）= 初期値推奨
#   Internal: 社内コラボ優先（内部拡散リスク増）
#   AnonymousAccess: 匿名リンク寄りで事故りやすい = 初期値非推奨
# - DefaultLinkPermission:
#   View : 閲覧既定（編集は意図したときだけ選ばせる）= 初期値推奨
#   Edit : 編集既定（誤共有時の破壊力が大きい）
$SPO_DefaultSharingLinkType = "Direct"
$SPO_DefaultLinkPermission  = "View"


# ==========================================================
# DO NOT EDIT: Derived Values（自動定義）
# ==========================================================
$TenantFallbackDomain = "$TenantOnMicrosoft.onmicrosoft.com"
$AdminUpn             = "$AdminUser@$TenantCustomDomain"

$SPOAdminUrl          = "https://$TenantOnMicrosoft-admin.sharepoint.com"
$SPORootUrl           = "https://$TenantOnMicrosoft.sharepoint.com"

$BreakGlassUpn1       = "$BreakGlassUser1@$TenantFallbackDomain"
$BreakGlassUpn2       = "$BreakGlassUser2@$TenantFallbackDomain"

# 共有メールボックスのアドレス（ドメイン追従）
$SharedMailbox_TechOps_Address  = "$SharedMailbox_TechOps_Alias@$TenantCustomDomain"
$SharedMailbox_SecOps_Address   = "$SharedMailbox_SecOps_Alias@$TenantCustomDomain"
$SharedMailbox_Privacy_Address  = "$SharedMailbox_Privacy_Alias@$TenantCustomDomain"

# Organization の Privacy contact email は命名一貫性のため Org系として保持
$OrgPrivacyContactEmail = $SharedMailbox_Privacy_Address


# ==========================================================
# Change Log（任意）
# ==========================================================
# 2026-02-17: v1.0 作成（初期構築パラメータシート化）

```

---

## Phase 1: 前提確認（目視）

```powershell
$PSVersionTable.PSVersion
whoami
```

---

## Phase 2: 接続テスト（設定変更なし）

### 2.1 Microsoft Graph

```powershell
# --- Disconnect ---
Disconnect-MgGraph -ErrorAction SilentlyContinue

# --- Connect ---
Connect-MgGraph `
  -TenantId $TenantFallbackDomain `
  -Scopes "RoleManagement.ReadWrite.Directory","Policy.ReadWrite.ConditionalAccess" `
  -UseDeviceCode `
  -ContextScope Process

# 接続確認
(Get-MgContext).Account
(Get-MgContext).TenantId
```

### 2.2 Exchange Online

```powershell
# --- Disconnect ---
Disconnect-ExchangeOnline -Confirm:$false -ErrorAction SilentlyContinue

# --- Connect（通常） ---
Connect-ExchangeOnline -UserPrincipalName $AdminUpn -ShowBanner:$false -ErrorAction SilentlyContinue

# テナント確認
Get-OrganizationConfig | Select-Object DisplayName

# 別アカウントに寄った場合は Device Code
if (-not $exoTenantId) {
  Disconnect-ExchangeOnline -Confirm:$false -ErrorAction SilentlyContinue
  Connect-ExchangeOnline -UserPrincipalName $AdminUpn -Device -ShowBanner:$false
  (Get-ConnectionInformation | Select-Object -First 1).TenantId
}
```

### 2.3 Microsoft Teams

```powershell
# --- Disconnect ---
Disconnect-MicrosoftTeams -ErrorAction SilentlyContinue

# --- Connect ---
Connect-MicrosoftTeams -AccountId $AdminUpn

# テナント確認
Get-CsTenant | Select-Object DisplayName
```

### 2.4 SharePoint Online（PS7 → WinPSCompatSession）

```powershell
# --- Import (必須：PS7では自動ロードされないことがある) ---
Import-Module Microsoft.Online.SharePoint.PowerShell -Force

# --- Disconnect ---
Disconnect-SPOService -ErrorAction SilentlyContinue

# --- Connect ---
Connect-SPOService -Url $SPOAdminUrl -UseSystemBrowser:$true

# テナント確認（SPO側の事実）
(Get-SPOSite -Identity $SPORootUrl -ErrorAction Stop) | Select-Object Url, Title
```

