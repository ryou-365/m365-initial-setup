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
$TenantCustomDomain = "example.com"                       # 独自ドメイン
$TenantOnMicrosoft  = "example-legacy123.onmicrosoft.com" # テナント作成時に指定したサブドメインを含めたフォールバックドメイン
$AdminUpn           = "admin@$TenantCustomDomain"         # 管理者UPN
# SharePoint Online
$SPOTenantPrefix    = "example-legacy123"                 # https://<prefix>.sharepoint.com の <prefix>
$SPOAdminUrl        = "https://$SPOTenantPrefix-admin.sharepoint.com" # URLはテナントの実値に合わせて修正する
$SPORootUrl         = "https://$SPOTenantPrefix.sharepoint.com"
# 緊急アクセス用アカウント
$BreakGlassUpn1     = "breakglass1@$TenantOnMicrosoft"
$BreakGlassUpn2     = "breakglass2@$TenantOnMicrosoft"
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
Disconnect-MgGraph -ErrorAction SilentlyContinue
Connect-MgGraph -Scopes "Organization.Read.All"
$tid_graph = (Get-MgOrganization).Id
$tid_graph
```

### 2.2 Exchange Online（WAM 回避を既定）

> 環境により WAM（Web Account Manager）関連例外が出るため、`-DisableWAM` を既定とします。

```powershell
Import-Module ExchangeOnlineManagement
Disconnect-ExchangeOnline -Confirm:$false -ErrorAction SilentlyContinue
Connect-ExchangeOnline -UserPrincipalName $AdminUpn -DisableWAM -ShowBanner:$false
Get-OrganizationConfig | Select-Object Name
```

### 2.3 Microsoft Teams

```powershell
Import-Module MicrosoftTeams
Disconnect-MicrosoftTeams -ErrorAction SilentlyContinue
Connect-MicrosoftTeams
$tid_teams = (Get-CsTenant).TenantId
$tid_teams
```

### 2.4 SharePoint Online（PS7 → WinPSCompatSession）

> 目的: **意図した SPO テナントに接続できているか**を、SPO 側の情報だけで確認する。

```powershell
Import-Module Microsoft.Online.SharePoint.PowerShell -UseWindowsPowerShell
Disconnect-SPOService -ErrorAction SilentlyContinue
Connect-SPOService -Url $SPOAdminUrl -UseSystemBrowser $true
Get-SPOSite -Limit 1 | Select-Object Url
```

合格条件（目視）:

* `Url` が `https://$SPOTenantPrefix.sharepoint.com/...` の形式で返る
* 返った `Url` のホスト名が `$SPORootUrl` と一致する

---

## 参考: TenantId の整合（Graph / Teams）

> 目的: **意図しない別テナントへの接続**を検知する（SPO/EXO ではなく、SoT として Graph/Teams を利用）。

```powershell
"$tid_graph / $tid_teams"
```

合格条件（目視）:

* TenantId が一致する
