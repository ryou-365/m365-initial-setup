# 01_EmergencyAccess

* 緊急アクセス（break-glass）確立 & ロックアウト耐性の確保

## ねらい

本章では、Microsoft 365 初期構築において **最優先で実施すべき安全装置**として、
緊急用アクセスアカウント（break-glass）を確立します。

* 管理者ロックアウトを防ぐための **最後の入口** を確保する
* 後続の条件付きアクセス（CA）や MFA 強制を **安全に適用できる前提** を作る
* 除外設計を構造化し、将来的なレビュー・変更耐性を持たせる

本章は **認証制御の強化を行う章ではありません**。
CA 本体・MFA 強制は README に定義された通り、後続の `04_ConditionalAccessBaseline` にて実施します。

---

## 位置づけ（README との対応）

* 区分: **Mandatory（やらないと危険）**
* 対象: `01_EmergencyAccess`
* 前提:

  * `00_Initialization` が完了していること
  * 管理用アカウントで Microsoft Graph に接続できること

---

# Phase 0: Tenant Variables（EDIT HERE ONLY）

```powershell
# カスタムドメイン
$TenantCustomDomain   = "example.com"

# onmicrosoft.com のサブドメイン
$TenantOnMicrosoft    = "example-fallback"

# 管理者アカウント名
$AdminUser            = "admin"

# 緊急アクセス用アカウント名（技術識別子）
$BreakGlassUser1      = "breakglass1"
$BreakGlassUser2      = "breakglass2"

# 緊急アクセス用表示名（役割明示）
$BreakGlassDisplayName1 = "BreakGlass Account 1"
$BreakGlassDisplayName2 = "BreakGlass Account 2"

# --- 自動定義（編集禁止） ---
$TenantFallbackDomain = "$TenantOnMicrosoft.onmicrosoft.com"
$AdminUpn             = "$AdminUser@$TenantCustomDomain"
$SPOAdminUrl          = "https://$TenantOnMicrosoft-admin.sharepoint.com"
$SPORootUrl           = "https://$TenantOnMicrosoft.sharepoint.com"

$BreakGlassUpn1       = "$BreakGlassUser1@$TenantFallbackDomain"
$BreakGlassUpn2       = "$BreakGlassUser2@$TenantFallbackDomain"

# break-glass 除外用グループ
$BreakGlassExemptGroupName = "sg-entra-breakglass-exempt"
```

---

# Phase 1: 前提確認・接続（Graph）

> 本章では **ユーザー / グループ / ロール操作のみ**を行います。

```powershell
# 接続コマンド: アカウント作成、グローバル管理者権限の付与を行うためのスコープを指定して接続
Connect-MgGraph -Scopes "User.ReadWrite.All","Directory.ReadWrite.All","RoleManagement.ReadWrite.Directory" -NoWelcome
(Get-MgContext | Select-Object TenantId, Account)
```

```powershell
# Global Administrator ロール取得
$gaRole = Get-MgDirectoryRole | Where-Object DisplayName -eq "Global Administrator"
```

```powershell
## イレギュラー対応用
### グローバル管理者のテンプレートID (固定値)
#$gaTemplateId = "62e90394-69f5-4237-9190-012177145e10"
### 役割がまだ有効化されていない場合に有効化
#$gaRole = Get-MgDirectoryRole | Where-Object DisplayName -eq "Global Administrator"
#if (-not $gaRole) {
#  Enable-MgDirectoryRole -RoleTemplateId $gaTemplateId
#  $gaRole = Get-MgDirectoryRole | Where-Object DisplayName -eq "Global Administrator"
#}
```

---

# Phase 2: break-glass 設計方針（確定事項）

* break-glass は **2アカウント**（冗長構成）
* **両アカウントとも条件付きアクセス（CA）から完全除外**
* 通常業務では使用しない

  * サインイン発生 = インシデント扱い
* 資格情報は Git 管理下に置かない

---

# Phase 3: break-glass アカウント作成（クラウド専用）

## 3.1 既存確認（目視）

```powershell
Get-MgUser -UserId $BreakGlassUpn1 -ErrorAction SilentlyContinue | Select DisplayName,UserPrincipalName,Id
```

```powershell
Get-MgUser -UserId $BreakGlassUpn2 -ErrorAction SilentlyContinue | Select DisplayName,UserPrincipalName,Id
```

## 3.2 作成（存在しない場合のみ）

> パスワードは **別経路で生成・保管**してください。

```powershell
# パスワードは Keeperなど からコピペ
$sec1 = Read-Host "Password for $BreakGlassUpn1 " -AsSecureString
$pw1  = [Runtime.InteropServices.Marshal]::PtrToStringBSTR(
          [Runtime.InteropServices.Marshal]::SecureStringToBSTR($sec1)
        )

# 作成
New-MgUser -BodyParameter @{
    displayName = $BreakGlassDisplayName1
    userPrincipalName = $BreakGlassUpn1
    mailNickname = $BreakGlassUser1
    accountEnabled = $true
    passwordProfile = @{
        password = $pw1
        forceChangePasswordNextSignIn = $false
    }
}
```

```powershell
# パスワードは Keeperなど からコピペ
$sec2 = Read-Host "Password for $BreakGlassUpn2 " -AsSecureString
$pw2  = [Runtime.InteropServices.Marshal]::PtrToStringBSTR(
          [Runtime.InteropServices.Marshal]::SecureStringToBSTR($sec2)
        )

# 作成
New-MgUser -BodyParameter @{
    displayName = $BreakGlassDisplayName2
    userPrincipalName = $BreakGlassUpn2
    mailNickname = $BreakGlassUser2
    accountEnabled = $true
    passwordProfile = @{
        password = $pw2
        forceChangePasswordNextSignIn = $false
    }
}
```

---

# Phase 4: Global Administrator ロール付与

```powershell
# GA 付与
$bgu1 = Get-MgUser -UserId $BreakGlassUpn1
New-MgDirectoryRoleMember -DirectoryRoleId $gaRole.Id -DirectoryObjectId $bgu1.Id
```

```powershell
# GA 付与
$bgu2 = Get-MgUser -UserId $BreakGlassUpn2
New-MgDirectoryRoleMember -DirectoryRoleId $gaRole.Id -DirectoryObjectId $bgu2.Id
```

---

# Phase 5: break-glass 除外グループ（動的）

## 5.1 方針

* CA 除外は **このグループにのみ集約**
* メンバーは **UPN 完全一致（2名）** の動的ルール
* 手動追加・削除を禁止し、**ルール変更のみをレビュー対象**とする

## 5.2 作成

```powershell
$rule = "(user.userPrincipalName -eq \"$BreakGlassUpn1\") -or (user.userPrincipalName -eq \"$BreakGlassUpn2\")"
```

```powershell
if (-not (Get-MgGroup -Filter "displayName eq '$BreakGlassExemptGroupName'")) {
  New-MgGroup -DisplayName $BreakGlassExemptGroupName `
    -MailEnabled:$false `
    -MailNickname "sg-entra-breakglass-exempt" `
    -SecurityEnabled:$true `
    -GroupTypes @("DynamicMembership") `
    -MembershipRule $rule `
    -MembershipRuleProcessingState "On"
}
```

---

# 完了条件（チェック）

* [ ] break-glass アカウントが **2名** 存在する
* [ ] 両方が **Global Administrator** である
* [ ] 動的除外グループに **2名のみ** 含まれている

