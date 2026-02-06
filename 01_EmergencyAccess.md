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
$TenantCustomDomain = "example.com"                       # 独自ドメイン
$TenantOnMicrosoft  = "example-legacy123.onmicrosoft.com" # フォールバックドメイン
$AdminUpn           = "admin@$TenantCustomDomain"         # 管理者UPN
# break-glass（2アカウント固定）
$BreakGlassUpn1     = "breakglass1@$TenantOnMicrosoft"
$BreakGlassUpn2     = "breakglass2@$TenantOnMicrosoft"
# break-glass 除外用グループ
$BreakGlassExemptGroupName = "sg-entra-breakglass-exempt"
```

---

# Phase 1: 前提確認・接続（Graph）

> 本章では **ユーザー / グループ / ロール操作のみ**を行います。

```powershell
Import-Module Microsoft.Graph
```

```powershell
Disconnect-MgGraph -ErrorAction SilentlyContinue
```

```powershell
Connect-MgGraph -Scopes @(
  "Organization.Read.All",
  "User.ReadWrite.All",
  "Group.ReadWrite.All",
  "RoleManagement.ReadWrite.Directory"
)
```

```powershell
Get-MgContext | Select-Object Account,TenantId,Scopes
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
$pwd1 = "<GENERATE-STRONG-PASSWORD-1>"
$pwd2 = "<GENERATE-STRONG-PASSWORD-2>"
```

```powershell
if (-not (Get-MgUser -UserId $BreakGlassUpn1 -ErrorAction SilentlyContinue)) {
  New-MgUser -AccountEnabled:$true `
    -DisplayName "BreakGlass 1" `
    -MailNickname "breakglass1" `
    -UserPrincipalName $BreakGlassUpn1 `
    -PasswordProfile @{ ForceChangePasswordNextSignIn = $false; Password = $pwd1 }
}
```

```powershell
if (-not (Get-MgUser -UserId $BreakGlassUpn2 -ErrorAction SilentlyContinue)) {
  New-MgUser -AccountEnabled:$true `
    -DisplayName "BreakGlass 2" `
    -MailNickname "breakglass2" `
    -UserPrincipalName $BreakGlassUpn2 `
    -PasswordProfile @{ ForceChangePasswordNextSignIn = $false; Password = $pwd2 }
}
```

---

# Phase 4: Global Administrator ロール付与

```powershell
$ga = Get-MgDirectoryRole | Where-Object DisplayName -eq "Global Administrator"
```

```powershell
if (-not $ga) {
  $tmpl = Get-MgDirectoryRoleTemplate | Where-Object DisplayName -eq "Global Administrator"
  New-MgDirectoryRole -RoleTemplateId $tmpl.Id | Out-Null
  $ga = Get-MgDirectoryRole | Where-Object DisplayName -eq "Global Administrator"
}
```

```powershell
$bg1 = Get-MgUser -UserId $BreakGlassUpn1
$bg2 = Get-MgUser -UserId $BreakGlassUpn2
```

```powershell
New-MgDirectoryRoleMember -DirectoryRoleId $ga.Id -DirectoryObjectId $bg1.Id
```

```powershell
New-MgDirectoryRoleMember -DirectoryRoleId $ga.Id -DirectoryObjectId $bg2.Id
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

