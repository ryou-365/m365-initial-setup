# PowerShellによるM365構築準備

## 最新の PowerShell 7 をインストールする

Windows 11標準の5.1ではなく、最新の7をインストールします。

```powershell
# PowerShell 7をコマンドでインストール（実行後、指示に従ってください）
winget install --id Microsoft.Powershell --source winget
```

> 注意: インストール完了後、一度すべてのウィンドウを閉じ、青いアイコン（5.1）ではなく黒いアイコンの「PowerShell 7」を右クリックして「管理者として実行」で開き直してください。

## モジュールのインストール

```powershell
# 実行ポリシーの変更（スクリプト実行を許可）
Set-ExecutionPolicy RemoteSigned -Force

# Microsoft Graph (基本管理)
Install-Module -Name Microsoft.Graph -AllowClobber -Force -Scope CurrentUser

# Exchange Online (メール管理)
Install-Module -Name ExchangeOnlineManagement -AllowClobber -Force -Scope CurrentUser

# Microsoft Teams (Teams管理)
Install-Module -Name MicrosoftTeams -AllowClobber -Force -Scope CurrentUser

# SharePoint Online (SharePoint/OneDrive管理)
Install-Module -Name PnP.PowerShell -AllowClobber -Force -Scope CurrentUser
```

## 接続コマンド

```powershell
# Microsoft Graphへの接続（必要な権限をスコープで指定）
Connect-MgGraph -Scopes "User.ReadWrite.All", "Group.ReadWrite.All", "Directory.ReadWrite.All", "Organization.Read.All"

# Exchange Onlineへの接続
Connect-ExchangeOnline
```

## 接続時の認証パターン解説

M365管理における認証は、以下の3つのパターンが主流です。

### ① インタラクティブ認証（ブラウザ認証）

**用途：** 初回の設定、単発の作業、検証環境。
最も一般的で、多要素認証（MFA）にも対応しています。

- **メリット：** 設定が不要。コマンドを打つだけでサインイン画面が出る。
- **デメリット：** スクリプトの自動実行（スケジュール実行）ができない。

```powershell
# 基本的な接続（ブラウザが起動し、ログインを求められる）

# Graphに接続（初回は承認画面が出ます）
Connect-MgGraph -Scopes "User.ReadWrite.All", "Group.ReadWrite.All", "Directory.ReadWrite.All", "Organization.Read.All"

# Exchange Onlineに接続（ご自身のメールアドレスに書き換えてください）
Connect-ExchangeOnline -UserPrincipalName "admin@example.com"

# Teamsに接続
Connect-MicrosoftTeams

# SharePoint Onlineに接続（ご自身のドメインに書き換えてください）
Connect-PnPOnline -Url "https://yourdomain-admin.sharepoint.com" -Interactive
```

### ② 証明書認証（無人実行 / 自動化）

**用途：** 定期的なバックアップ、ユーザーの一括作成、自動設定。
**ゼロトラストにおける推奨:** ID/パスワードを使わず、秘密鍵を持つサーバーのみに実行権限を与える方式です。

- **準備：** Entra ID（旧Azure AD）に「アプリの登録」を行い、証明書をアップロードします。
    - ステップ1：自分のPCで証明書を作成する。
        
        ```powershell
        # 自己署名証明書の作成（有効期限2年）
        $cert = New-SelfSignedCertificate -DnsName "M365AdminAutomation" -CertStoreLocation "Cert:\CurrentUser\My" -NotAfter (Get-Date).AddYears(2)
        
        # M365登録用に、公開鍵（.cerファイル）を書き出す
        Export-Certificate -Cert $cert -FilePath "C:\temp\M365Admin.cer"
        ```
        
    - ステップ2：M365（Entra ID）にアップロードする。
        1. [Entra 管理センター](https://www.google.com/url?sa=E&source=gmail&q=https://entra.microsoft.com/) ＞ アプリの登録 ＞ 自分のアプリを選択。
        2. **「証明書とシークレット」** ＞ **「証明書のアップロード」** で、先ほど書き出した `.cer` ファイルを選択します。
- **メリット：** MFAをスキップして安全に自動実行が可能。

```powershell
# 事前定義（一例）
$TenantId = "your-tenant-id"
$AppId = "your-app-id"
$Thumbprint = "your-certificate-thumbprint"
$Domain = "yourdomain.onmicrosoft.com"

# --- 接続コマンド ---
Connect-MgGraph -TenantId $TenantId -AppId $AppId -CertificateThumbprint $Thumbprint
Connect-ExchangeOnline -CertificateThumbprint $Thumbprint -AppId $AppId -Organization $Domain
Connect-MicrosoftTeams -CertificateThumbprint $Thumbprint -ApplicationId $AppId -TenantId $TenantId
Connect-PnPOnline -Url "https://$($Domain.Split('.')[0])-admin.sharepoint.com" -ClientId $AppId -Thumbprint $Thumbprint -Tenant $TenantId
```

### ③ マネージドID（Azure VM/Automation上での実行）

**用途：** Azure上でスクリプトを動かす場合。
認証情報をコードや証明書として管理する必要すらなく、Azureのリソース自体に権限を付与します。

```powershell
# 1. 変数の準備（ご自身のドメインに書き換えて実行してください）
$Domain = "yourdomain.onmicrosoft.com"

# 2. Microsoft Graph に接続
Connect-MgGraph -Identity

# 3. Exchange Online に接続
Connect-ExchangeOnline -ManagedIdentity -Organization $Domain

# 4. Microsoft Teams に接続
# ※マネージドIDの「クライアントID」を $ManagedIdentityId に指定します
$ManagedIdentityId = "00000000-0000-0000-0000-000000000000"
Connect-MicrosoftTeams -Identity -AccountId $ManagedIdentityId

# 5. SharePoint Online / PnP に接続
# ※URLをご自身の環境に合わせて書き換えてください
$AdminUrl = "https://yourdomain-admin.sharepoint.com"
Connect-PnPOnline -Url $AdminUrl -ManagedIdentity
```
