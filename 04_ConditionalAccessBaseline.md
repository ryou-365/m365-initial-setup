# 04_ConditionalAccessBaseline

* 条件付きアクセス（CA）を用いた、全体的な認証制御とゼロトラスト設計の適用（段階導入）

## ねらい

Microsoft 365 Business Premium（Entra ID P1）前提で、
**Secure by default** を実現する最小構成の条件付きアクセスを構築する。

本章ではまず、

* Office 365（Exchange / SharePoint / OneDrive / Teams）を保護対象とする
* レガシー認証を全面遮断する
* 全員にMFAを要求する
* 管理対象デバイス（準拠端末）を前提とする
* ゲストはMFA必須とする

というベースラインを確立する。

管理系アプリ（Azure管理ポータル等）への拡張は後段で行う。

---

## 位置づけ（READMEとの対応）

* 区分: Recommended
* 対象: `04_ConditionalAccessBaseline`
* 前提:

  * `00_Initialization` 完了
  * `01_EmergencyAccess` 完了
  * `02_AuditAndBaseline` 完了
  * `03_TenantCoreSettings` 完了

---

# Phase 0: Tenant Variables（EDIT HERE ONLY）

> 00_Initialization の Phase 0 を参照（再掲しない）

- [00_Initialization.md #phase-0-tenant-variables](https://github.com/ryou-365/m365-initial-setup/blob/main/00_Initialization.md#phase-0-tenant-variablesedit-here-only)

---

# Phase 1: Microsoft Graph 接続

```powershell
Disconnect-MgGraph -ErrorAction SilentlyContinue

Connect-MgGraph `
  -TenantId $TenantFallbackDomain `
  -Scopes "Policy.ReadWrite.ConditionalAccess","Group.ReadWrite.All","Directory.Read.All" `
  -UseDeviceCode `
  -ContextScope Process

(Get-MgContext).Account
(Get-MgContext).TenantId
```

---

# Phase 2: CA用グループ作成（冪等）

> 本章では CA の include/exclude に使う最小限のグループだけを用意します。
>
> * break-glass 除外グループは `01_EmergencyAccess` で作成済み前提
> * 特権ユーザーは **手動メンバー**（運用で厳格に管理）
> * ゲストは **動的グループ**（userType=Guest）

## 2.0 break-glass 除外グループID取得（必須）

```powershell
$bg = Get-MgGroup -Filter "displayName eq '$BreakGlassExemptGroupName'" -ConsistencyLevel eventual -ErrorAction Stop
$BreakGlassExemptGroupId = $bg.Id
$BreakGlassExemptGroupId
```

## 2.1 特権ユーザー（手動管理）

```powershell
$privGroup = Get-MgGroup -Filter "displayName eq '$CA_PrivilegedGroupName'" -ConsistencyLevel eventual

if (-not $privGroup) {
  $privGroup = New-MgGroup -BodyParameter @{
    displayName     = $CA_PrivilegedGroupName
    mailEnabled     = $false
    mailNickname    = $CA_PrivilegedGroupMailNick
    securityEnabled = $true
  }
}

$privGroup.Id
```

## 2.2 ゲスト（動的グループ）

```powershell
$guestGroup = Get-MgGroup -Filter "displayName eq '$CA_GuestGroupName'" -ConsistencyLevel eventual

if (-not $guestGroup) {
  $guestGroup = New-MgGroup -BodyParameter @{
    displayName                    = $CA_GuestGroupName
    mailEnabled                    = $false
    mailNickname                   = $CA_GuestGroupMailNick
    securityEnabled                = $true
    groupTypes                     = @("DynamicMembership")
    membershipRule                 = $CA_GuestDynamicRule
    membershipRuleProcessingState  = "On"
  }
} else {
  # ルール差分があれば更新（冪等）
  if ($guestGroup.MembershipRule -ne $CA_GuestDynamicRule) {
    Update-MgGroup -GroupId $guestGroup.Id -BodyParameter @{
      membershipRule                = $CA_GuestDynamicRule
      membershipRuleProcessingState = "On"
    }
    $guestGroup = Get-MgGroup -GroupId $guestGroup.Id
  }
}

$guestGroup.Id
```

## 2.3 例外（任意：一時例外のみ）

> 恒久的な穴にしない運用前提。

```powershell
$excGroup = Get-MgGroup -Filter "displayName eq '$CA_ExceptionGroupName'" -ConsistencyLevel eventual

if (-not $excGroup) {
  $excGroup = New-MgGroup -BodyParameter @{
    displayName     = $CA_ExceptionGroupName
    mailEnabled     = $false
    mailNickname    = $CA_ExceptionGroupMailNick
    securityEnabled = $true
  }
}

$excGroup.Id
```

---

# Phase 3: 条件付きアクセスポリシー作成（Office 365限定）

## 3.x 作成／更新の方針（関数は使わない）

本章では、各ポリシーについて **「存在確認 → なければ作成 / あれば更新」** を `if` で直書きします。
（見た目は長くなるが、PowerShellコマンド＋条件分岐だけで完結させるため）

---

## L001_全員_レガシー認証をブロック（All apps）

```powershell
$body = @{
  displayName = $CA_L001_Name
  state       = $CA_InitialState
  conditions  = @{
    users = @{
      includeUsers  = @("All")
      excludeGroups = @($BreakGlassExemptGroupId)
    }
    applications = @{
      includeApplications = @("All")
    }
    clientAppTypes = @("exchangeActiveSync","other")
  }
  grantControls = @{
    operator        = "OR"
    builtInControls = @("block")
  }
}

# 既存確認
$existing = Get-MgIdentityConditionalAccessPolicy -Filter "displayName eq '$CA_L001_Name'" | Select-Object -First 1

# なければ作成、あれば更新（冪等）
if (-not $existing) {
  New-MgIdentityConditionalAccessPolicy -BodyParameter $body | Out-Null
  Write-Host "Created: $CA_L001_Name"
} else {
  Update-MgIdentityConditionalAccessPolicy -ConditionalAccessPolicyId $existing.Id -BodyParameter $body | Out-Null
  Write-Host "Updated: $CA_L001_Name"
}
```

---

## L002_全員_MFA必須（Office 365）

```powershell
$body = @{
  displayName = $CA_L002_Name
  state       = $CA_InitialState
  conditions  = @{
    users = @{
      includeUsers  = @("All")
      excludeGroups = @($BreakGlassExemptGroupId)
    }
    applications = @{
      includeApplications = $CA_Include_Applications
    }
  }
  grantControls = @{
    operator        = "OR"
    builtInControls = @("mfa")
  }
}

# 既存確認
$existing = Get-MgIdentityConditionalAccessPolicy -Filter "displayName eq '$CA_L002_Name'" | Select-Object -First 1

# なければ作成、あれば更新（冪等）
if (-not $existing) {
  New-MgIdentityConditionalAccessPolicy -BodyParameter $body | Out-Null
  Write-Host "Created: $CA_L002_Name"
} else {
  Update-MgIdentityConditionalAccessPolicy -ConditionalAccessPolicyId $existing.Id -BodyParameter $body | Out-Null
  Write-Host "Updated: $CA_L002_Name"
}
```

---

## R001_一般_準拠デバイス必須

```powershell
$body = @{
  displayName = $CA_R001_Name
  state       = $CA_InitialState
  conditions  = @{
    users = @{
      includeUsers  = @("All")
      excludeGroups = @($privGroup.Id, $guestGroup.Id, $BreakGlassExemptGroupId)
    }
    applications = @{
      includeApplications = $CA_Include_Applications
    }
  }
  grantControls = @{
    operator        = "OR"
    builtInControls = @("compliantDevice")
  }
}

# 既存確認
$existing = Get-MgIdentityConditionalAccessPolicy -Filter "displayName eq '$CA_R001_Name'" | Select-Object -First 1

# なければ作成、あれば更新（冪等）
if (-not $existing) {
  New-MgIdentityConditionalAccessPolicy -BodyParameter $body | Out-Null
  Write-Host "Created: $CA_R001_Name"
} else {
  Update-MgIdentityConditionalAccessPolicy -ConditionalAccessPolicyId $existing.Id -BodyParameter $body | Out-Null
  Write-Host "Updated: $CA_R001_Name"
}
```

---

## P001_特権_準拠デバイス必須

```powershell
$body = @{
  displayName = $CA_P001_Name
  state       = $CA_InitialState
  conditions  = @{
    users = @{
      includeGroups = @($privGroup.Id)
      excludeGroups = @($BreakGlassExemptGroupId)
    }
    applications = @{
      includeApplications = $CA_Include_Applications
    }
  }
  grantControls = @{
    operator        = "OR"
    builtInControls = @("compliantDevice")
  }
}

# 既存確認
$existing = Get-MgIdentityConditionalAccessPolicy -Filter "displayName eq '$CA_P001_Name'" | Select-Object -First 1

# なければ作成、あれば更新（冪等）
if (-not $existing) {
  New-MgIdentityConditionalAccessPolicy -BodyParameter $body | Out-Null
  Write-Host "Created: $CA_P001_Name"
} else {
  Update-MgIdentityConditionalAccessPolicy -ConditionalAccessPolicyId $existing.Id -BodyParameter $body | Out-Null
  Write-Host "Updated: $CA_P001_Name"
}
```

---

## G001_ゲスト_MFA必須

```powershell
$body = @{
  displayName = $CA_G001_Name
  state       = $CA_InitialState
  conditions  = @{
    users = @{
      includeGroups = @($guestGroup.Id)
      excludeGroups = @($BreakGlassExemptGroupId)
    }
    applications = @{
      includeApplications = $CA_Include_Applications
    }
  }
  grantControls = @{
    operator        = "OR"
    builtInControls = @("mfa")
  }
}

# 既存確認
$existing = Get-MgIdentityConditionalAccessPolicy -Filter "displayName eq '$CA_G001_Name'" | Select-Object -First 1

# なければ作成、あれば更新（冪等）
if (-not $existing) {
  New-MgIdentityConditionalAccessPolicy -BodyParameter $body | Out-Null
  Write-Host "Created: $CA_G001_Name"
} else {
  Update-MgIdentityConditionalAccessPolicy -ConditionalAccessPolicyId $existing.Id -BodyParameter $body | Out-Null
  Write-Host "Updated: $CA_G001_Name"
}
```

---

# Phase 4: 状態確認

```powershell
Get-MgIdentityConditionalAccessPolicy |
  Where-Object { $_.DisplayName -like "L*" -or $_.DisplayName -like "R*" -or $_.DisplayName -like "P*" -or $_.DisplayName -like "G*" } |
  Select-Object DisplayName, State |
  Sort-Object DisplayName
```

---

# 推奨導入順序

1. L001（レガシー遮断）
2. L002（全員MFA）
3. P001（特権準拠端末）
4. R001（一般準拠端末）
5. G001（ゲストMFA）

---

# 完了条件

* 5つのポリシーが存在する
* 初期状態は reportOnly
* BreakGlass が全ポリシーで除外されている
* ゲストが動的グループに正しく含まれる

---

# 補足

本章は「データ平面（Office 365）」の保護に限定する。
管理プレーン（Azure管理ポータル等）は成熟後に追加する。
