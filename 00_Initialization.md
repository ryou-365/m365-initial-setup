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

## Phase 0: Tenant Variables（EDIT HERE ONLY）

```powershell
# カスタムドメイン
$TenantCustomDomain   = "example.com"
# onmicrosoft.comのサブドメイン
$TenantOnMicrosoft    = "example-Fallback"

# 管理者アカウント名
$AdminUser            = "admin"

# 緊急アクセス用アカウント名
$BreakGlassUser1      = "breakglass1"
$BreakGlassUser2      = "breakglass2"

# 自動定義 (編集禁止)
$TenantFallbackDomain = "$TenantOnMicrosoft.onmicrosoft.com"
$AdminUpn             = "$AdminUser@$TenantCustomDomain"
$SPOAdminUrl          = "https://$TenantOnMicrosoft-admin.sharepoint.com"
$SPORootUrl           = "https://$TenantOnMicrosoft.sharepoint.com"
$BreakGlassUpn1       = "$BreakGlassUser1@$TenantFallbackDomain"
$BreakGlassUpn2       = "$BreakGlassUser2@$TenantFallbackDomain"
```

---

## Phase 1: 前提確認（目視）

```powershell
$PSVersionTable.PSVersion
whoami
```

---

## Phase 2: 接続テスト（設定変更なし）

### 2.1 Microsoft Graph（Read-only）

```powershell
Import-Module Microsoft.Graph
Connect-MgGraph -Scopes "User.Read" -NoWelcome
(Get-MgContext | Select-Object TenantId, Account)
```

### 2.2 Exchange Online（WAM 回避を既定）

```powershell
Import-Module ExchangeOnlineManagement
Connect-ExchangeOnline -UserPrincipalName $AdminUpn -ShowBanner:$false
(Get-ConnectionInformation | Select-Object -First 1).TenantId
```

### 2.3 Microsoft Teams

```powershell
Import-Module MicrosoftTeams
Connect-MicrosoftTeams -AccountId $AdminUpn
(Get-CsTenant).Identity
```

### 2.4 SharePoint Online（PS7 → WinPSCompatSession）

```powershell
Import-Module Microsoft.Online.SharePoint.PowerShell -UseWindowsPowerShell
Connect-SPOService -Url $SPOAdminUrl -UseSystemBrowser:$true

# 意図したテナントのサイトをSPO側から取得できるかで確認（SPO側の事実）
(Get-SPOSite -Identity $SPORootUrl -ErrorAction Stop) | Select-Object Url, Title
```

