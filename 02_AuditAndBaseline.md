# 02_AuditAndBaseline

* 監査ログ有効化 & EXO監査の最低限整備

## ねらい

本章では、Microsoft 365 初期構築において
**「後から説明できる状態」を作るための監査基盤」**を確立します。

* Unified Audit Log（UAL）を有効化する
* Exchange Online の管理操作監査を有効化する
* Mailbox Audit を有効化し、既存テナントの是正を行う

本章は **防御強化の章ではありません。**
メールフロー制御やレガシー認証遮断などの強化は
`05_ServiceSecurityBaseline` にて実施します。

---

## 位置づけ（README との対応）

* 区分: **Mandatory（証跡が残らない状態は危険）**
* 対象: `02_AuditAndBaseline`
* 前提:

  * `00_Initialization` が完了していること
  * `01_EmergencyAccess` が完了していること
  * 管理用アカウントで Exchange Online に接続できること

---

# Phase 0: Tenant Variables（EDIT HERE ONLY）

- [00_Initialization.md #phase-0-tenant-variablesedit-here-only](https://github.com/ryou-365/m365-initial-setup/blob/main/00_Initialization.md#phase-0-tenant-variablesedit-here-only)

---

# Phase 1: 前提確認・接続（Exchange Online）

> 本章では Exchange Online の監査設定のみ変更します。

```powershell
# --- Disconnect ---
Disconnect-ExchangeOnline -Confirm:$false -ErrorAction SilentlyContinue

# --- Connect ---
Connect-ExchangeOnline -UserPrincipalName $AdminUpn -ShowBanner:$false -ErrorAction SilentlyContinue

# テナント確認
Get-OrganizationConfig | Select-Object DisplayName

# 別アカウントに寄った場合は Device Code
$exoTenantId = (Get-ConnectionInformation | Select-Object -First 1).TenantId
if (-not $exoTenantId) {
  Disconnect-ExchangeOnline -Confirm:$false -ErrorAction SilentlyContinue
  Connect-ExchangeOnline -UserPrincipalName $AdminUpn -Device -ShowBanner:$false
  (Get-ConnectionInformation | Select-Object -First 1).TenantId
}
```

---

# Phase 2: Unified Audit Log（UAL）有効化

## 2.1 現状確認

```powershell
$adminAudit = Get-AdminAuditLogConfig
$adminAudit | Select-Object UnifiedAuditLogIngestionEnabled
```

## 2.2 無効なら有効化（冪等）

```powershell
if (-not $adminAudit.UnifiedAuditLogIngestionEnabled) {
  Set-AdminAuditLogConfig -UnifiedAuditLogIngestionEnabled $true
}

# 再確認
Get-AdminAuditLogConfig | Select-Object UnifiedAuditLogIngestionEnabled
```

---

# Phase 3: Admin Audit Log 有効化（管理操作監査）

## 3.1 現状確認

```powershell
Get-AdminAuditLogConfig | Select-Object AdminAuditLogEnabled
```

## 3.2 無効なら有効化（冪等）

```powershell
if (-not (Get-AdminAuditLogConfig).AdminAuditLogEnabled) {
  Set-AdminAuditLogConfig -AdminAuditLogEnabled $true
}

# 再確認
Get-AdminAuditLogConfig | Select-Object AdminAuditLogEnabled
```

---

# Phase 4: Mailbox Audit 有効化

## 4.1 組織設定確認

```powershell
Get-OrganizationConfig | Select-Object AuditDisabled
```

## 4.2 無効なら有効化（冪等）

```powershell
if ((Get-OrganizationConfig).AuditDisabled -eq $true) {
  Set-OrganizationConfig -AuditDisabled $false
}

# 再確認
Get-OrganizationConfig | Select-Object AuditDisabled
```

---

## 4.3 既存メールボックスの監査是正（既存テナント対策）

```powershell
Get-Mailbox -ResultSize Unlimited |
  Where-Object { $_.AuditEnabled -eq $false } |
  Set-Mailbox -AuditEnabled $true
```

---

# Phase 5: Unified Audit Log 動作確認（少量）

> 有効化直後はログが流れ始めるまで時間差がある場合があります。

```powershell
$start = (Get-Date).AddHours(-4)
$end   = Get-Date

Search-UnifiedAuditLog -StartDate $start -EndDate $end -ResultSize 10 |
  Select-Object -First 10
```

---

# Phase 6: 切断

```powershell
Disconnect-ExchangeOnline -Confirm:$false -ErrorAction SilentlyContinue
```

---

# 完了条件（チェック）

* [ ] UnifiedAuditLogIngestionEnabled = True
* [ ] AdminAuditLogEnabled = True
* [ ] AuditDisabled = False
* [ ] Search-UnifiedAuditLog が実行可能である

---

# 本章の位置づけ（重要）

02は「攻める章」ではありません。

ここで行っているのは、

> **“これ以降の構築作業を、後から説明できる状態にすること”**

です。

実際のセキュリティ強化は `04_ConditionalAccessBaseline` および
`05_ServiceSecurityBaseline` にて段階的に適用します。
