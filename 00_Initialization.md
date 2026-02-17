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


## Phase 0: Tenant Variables（EDIT HERE ONLY）

- 下記をコピーして手元で自身の環境に応じた設定を記述し、接続前にPowerShellで実行してください。

```powershell
# ==============================
# Tenant Core（必須）
# ==============================

# カスタムドメイン（例: example.com）
$TenantCustomDomain   = "example.com"

# onmicrosoft.com のサブドメイン（例: example-fallback → example-fallback.onmicrosoft.com）
$TenantOnMicrosoft    = "example-fallback"

# 管理者アカウント名（例: admin → admin@example.com）
$AdminUser            = "admin"

# ==============================
# Emergency Access（break-glass）
# ==============================

# 緊急アクセス用アカウント名（技術識別子）
$BreakGlassUser1      = "breakglass1"
$BreakGlassUser2      = "breakglass2"

# 緊急アクセス用表示名（役割明示）
$BreakGlassDisplayName1 = "BreakGlass Account 1"
$BreakGlassDisplayName2 = "BreakGlass Account 2"

# ==============================
# 自動定義（編集禁止）
# ==============================

$TenantFallbackDomain = "$TenantOnMicrosoft.onmicrosoft.com"
$AdminUpn             = "$AdminUser@$TenantCustomDomain"

$SPOAdminUrl          = "https://$TenantOnMicrosoft-admin.sharepoint.com"
$SPORootUrl           = "https://$TenantOnMicrosoft.sharepoint.com"

$BreakGlassUpn1       = "$BreakGlassUser1@$TenantFallbackDomain"
$BreakGlassUpn2       = "$BreakGlassUser2@$TenantFallbackDomain"

# break-glass 除外用グループ（CA で除外対象を集約するため）
$BreakGlassExemptGroupName     = "BreakGlass Exempt (CA Exclusion)"
$BreakGlassExemptGroupMailNick = "sg-entra-breakglass-exempt"

# ==============================
# 03_TenantCoreSettings: 通知用 共有メールボックス定義
# ==============================
# ※ ここで定義するアドレスは「テナント内の共有メールボックス」を前提とする
# ※ 外部通知は「共有メールボックス → 外部自動転送」で担保する（外部ユーザーに権限は付与しない）

# 技術運用通知用 共有メールボックス
$SharedMailbox_TechOps_DisplayName = "IT Operations"
$SharedMailbox_TechOps_Alias       = "it-ops"
$SharedMailbox_TechOps_Address     = "it-ops@$TenantCustomDomain"

# セキュリティ通知用 共有メールボックス
$SharedMailbox_SecOps_DisplayName  = "Security Operations"
$SharedMailbox_SecOps_Alias        = "sec-ops"
$SharedMailbox_SecOps_Address      = "sec-ops@$TenantCustomDomain"

# プライバシー窓口用 共有メールボックス
$SharedMailbox_Privacy_DisplayName = "Privacy Contact"
$SharedMailbox_Privacy_Alias       = "privacy"
$SharedMailbox_Privacy_Address     = "privacy@$TenantCustomDomain"

# ==============================
# 03_TenantCoreSettings: 内部メンバー（共有メールボックスへ付与）
# ==============================
# ※ 各共有メールボックスごとに個別定義する（職務分掌が分かれる前提）
# ※ ここには「テナント内ユーザーのUPN」のみを指定する

$SharedMailbox_TechOps_InternalMembers = @(
  $AdminUpn
)

$SharedMailbox_SecOps_InternalMembers = @(
  $AdminUpn
)

$SharedMailbox_Privacy_InternalMembers = @(
  $AdminUpn
)

# ==============================
# 03_TenantCoreSettings: 外部転送先（テナント障害時の独立経路）
# ==============================
# ※ 各共有メールボックスごとに個別定義する（外部委託/SOC/顧問などを想定）
# ※ ここにはテナント外メールアドレスを指定する

$SharedMailbox_TechOps_ExternalForwarding = @(
  "external-tech@example-external.net"
)

$SharedMailbox_SecOps_ExternalForwarding = @(
  "external-sec@example-external.net"
)

$SharedMailbox_Privacy_ExternalForwarding = @(
  "external-privacy@example-external.net"
)

# ==============================
# 03_TenantCoreSettings: Organization Profile 設定（Graph）
# ==============================
# ※ Microsoft Graph の Organization（組織）オブジェクトに設定する値
# ※ Tech/Sec の通知先は共有メールボックスを直接設定するため「専用変数」は持たない（二重管理回避）
# ※ Privacy は外部提示情報として URL を必ず持つ

# 既定言語（例: ja-JP）
$OrgPreferredLanguage = "ja-JP"

# プライバシー（外部提示用）
# ※ 命名一貫性のため Org 系変数として保持し、共有メールボックスのアドレスを格納する
$OrgPrivacyContactEmail = $SharedMailbox_Privacy_Address
$OrgPrivacyStatementUrl = "https://$TenantCustomDomain/privacy"

# ==============================
# 03_TenantCoreSettings: SharePoint/OneDrive 共有リンク既定（SPO）
# ==============================
# これは「共有できる範囲（外部共有可否など）」ではなく、
# ユーザーが共有ボタンを押したときの“デフォルトの作られ方”を決める設定。

# 選定基準（初期値の考え方）:
# - Direct: 共有対象（相手）を明示する方向に寄せ、リンク流出時の事故を起こしにくくする
# - Internal: 社内コラボ優先（社内拡散リスクは上がる）
# - AnonymousAccess: 匿名リンク寄りになり事故りやすいので初期値には非推奨
#
# - View: 閲覧を既定にして、編集は意図したときだけ選ばせる
# - Edit: 編集が既定になり誤共有時の破壊力が大きい

# DefaultSharingLinkType: Direct / Internal / AnonymousAccess
$SPO_DefaultSharingLinkType = "Direct"

# DefaultLinkPermission: View / Edit
$SPO_DefaultLinkPermission  = "View"

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

