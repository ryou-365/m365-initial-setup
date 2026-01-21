# ゼロトラスト・ベストプラクティス設定

## 緊急用アクセスアカウントの作成

```powershell
# アカウント情報の定義
$UPN = "emergency-access-account@<ご自身のドメイン>.onmicrosoft.com"
$DisplayName = "緊急用管理者"
$Pass = "Complex-Password-Safe"

# ユーザー作成
$User = New-MgUser -DisplayName $DisplayName1 -UserPrincipalName $UPN -MailNickname "emergency" -AccountEnabled $true -PasswordProfile @{ Password = $Pass; ForceChangePasswordNextSignIn = $false }

# 「全体管理者」権限の付与
$Role = Get-MgDirectoryRole | Where-Object DisplayName -eq "Global Administrator"
New-MgDirectoryRoleMember -DirectoryRoleId $Role.Id -DirectoryObjectMemberId $User.Id
```

## セキュリティの既定値群の確認（PowerShell）

```powershell
# セキュリティの既定値群の設定状態を確認（IsAppliedがTrueなら有効）
Get-MgBetaPolicyFeatureRolloutPolicy -FeatureRolloutPolicyId "DirectorySetting" | Select-Object Id, IsApplied

# もし無効(False)で、かつ条件付きアクセスを使わないなら有効化する
Update-MgBetaPolicyFeatureRolloutPolicy -FeatureRolloutPolicyId "DirectorySetting" -IsApplied $true

# もし有効(True)で、かつ条件付きアクセスを使うなら無効化する
Update-MgBetaPolicyFeatureRolloutPolicy -FeatureRolloutPolicyId "DirectorySetting" -IsApplied $false
```

## レガシー認証のブロック（Exchange Online）

```powershell
# 組織全体でレガシー認証をブロック
Set-OrganizationConfig -DefaultAuthenticationPolicy "BlockLegacyAuth"
```

## 全ユーザーへのMFA強制＆緊急用アカウント除外（条件付きアクセス）

### ポリシーの作成(レポートモード)

```powershell
# 緊急用アカウントのIDを取得
$Emergency = Get-MgUser -Filter "UserPrincipalName eq 'emergency-access-account@<ドメイン>.onmicrosoft.com'"

# ポリシーの構成内容を定義
$Params = @{
    DisplayName = "CA001: 全ユーザーへのMFA強制（緊急用アカウント除外）"
    State = "EnabledForReportingButNotEnforced" # レポートモード
    Conditions = @{
        Users = @{
            IncludeUsers = @("All") # 全ユーザーが対象
            ExcludeUsers = @($Emergency.Id) # 緊急用アカウントのみ除外
        }
        Applications = @{
            IncludeApplications = @("All") # すべてのクラウドアプリが対象
        }
    }
    GrantControls = @{
        Operator = "OR"
        BuiltInControls = @("Mfa") # MFAを要求
    }
}

# ポリシーの作成
New-MgIdentityConditionalAccessPolicy -BodyParameter $Param
```

### ポリシーの有効化

```powershell
# 名前が一致するポリシーを探して情報を取得
$TargetPolicy = Get-MgIdentityConditionalAccessPolicy | Where-Object { $_.DisplayName -eq "CA001: 全ユーザーへのMFA強制（緊急用アカウント除外）" }

# IDが正しく取得できたか確認（画面にIDが表示されればOK）
$TargetPolicy.Id

# 状態を Enabled に書き換えるための設定値
$UpdateParams = @{State = "Enabled"}

# ポリシーの更新を実行
Update-MgIdentityConditionalAccessPolicy -ConditionalAccessPolicyId $TargetPolicy.Id -BodyParameter $UpdateParams

# ポリシー名と現在の状態を表示
Get-MgIdentityConditionalAccessPolicy | Select-Object DisplayName, State
```
