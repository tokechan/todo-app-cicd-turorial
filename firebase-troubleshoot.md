# Firebase CICD トラブルシューティング記録

## 概要
Todo アプリケーションを GitHub Actions 経由で Firebase Hosting に自動デプロイするための CICD パイプライン構築時に発生した問題と解決策の記録。

## プロジェクト構成
- **アプリケーション**: React + TypeScript + Vite
- **テスト**: Vitest
- **ホスティング**: Firebase Hosting
- **CICD**: GitHub Actions
- **Firebase プロジェクト**: `cicd-todo-app-89c3b`

## 発生した問題と解決策

### 1. Firebase サービスアカウント認証の問題

#### 問題
- GitHub Actions で Firebase にデプロイする際の認証エラー
- サービスアカウントキーの形式に関する問題

#### 初期状態
```yaml
- name: Prepeare Google Application Credentials
  run: |
    printf '%s' "${{ secrets.FIREBASE_SERVICE_ACCOUNT_B64 }}" > "$RUNNER_TEMP/key.json"
    echo "GOOGLE_APPLICATION_CREDENTIALS=$RUNNER_TEMP/key.json" >> $GITHUB_ENV
```

#### 解決策
サービスアカウントキーの処理を堅牢化し、JSON形式とBase64形式の両方に対応：

```yaml
- name: Prepare Google Application Credentials (robust)
  run: |
    RAW="$RUNNER_TEMP/cred.raw"
    printf '%s' '${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}' > "$RAW"

    # 先頭1文字が { なら生JSON、それ以外は base64 とみなして復号
    if head -c1 "$RAW" | grep -q '{'; then
      mv "$RAW" "$RUNNER_TEMP/key.json"
    else
      base64 -d "$RAW" > "$RUNNER_TEMP/key.json" || {
        echo "Secret is neither valid JSON nor base64. Recheck FIREBASE_SERVICE_ACCOUNT_JSON."; exit 1;
      }
    fi

    echo "GOOGLE_APPLICATION_CREDENTIALS=$RUNNER_TEMP/key.json" >> "$GITHUB_ENV"
```

#### 変更点
- シークレット名を `FIREBASE_SERVICE_ACCOUNT_B64` から `FIREBASE_SERVICE_ACCOUNT_JSON` に変更
- JSON形式とBase64形式の自動判定機能を追加
- エラーハンドリングの強化

### 2. Firebase プロジェクト指定の問題

#### 問題
- デプロイ時にプロジェクトが適切に指定されていない
- `webframeworks` 実験機能の不要な有効化

#### 初期状態
```yaml
- name: Deploy Firebase
  run: |
    firebase experiments:enable webframeworks
    firebase deploy --only hosting --non-interactive
```

#### 解決策
```yaml
- name: Deploy Firebase
  run: |
    firebase deploy --only hosting --project ${{ secrets.FIREBASE_PROJECT_ID }} --non-interactive
```

#### 変更点
- `webframeworks` 実験機能の削除（不要）
- `--project` フラグでプロジェクトを明示的に指定
- `FIREBASE_PROJECT_ID` シークレットの使用

### 3. サービスアカウント検証の追加

#### 追加された機能
```yaml
- name: Validate service account JSON & project
  run: |
    sudo apt-get update && sudo apt-get install -y jq >/dev/null
    test -s "$RUNNER_TEMP/key.json" || { echo "empty key.json (secret missing)"; exit 1; }
    echo "SA project_id: $(jq -r '.project_id' "$RUNNER_TEMP/key.json")"
    echo "SA type      : $(jq -r '.type' "$RUNNER_TEMP/key.json")"
```

#### 目的
- サービスアカウントファイルの有効性を確認
- プロジェクトIDとサービスアカウントタイプの表示
- デバッグ情報の提供

### 4. Firebase設定ファイルの調整

#### `.firebaserc` の設定
```json
{
  "projects": {
    "default": "cicd-todo-app-89c3b"
  }
}
```

#### `firebase.json` の設定
```json
{
  "hosting": {
    "public": "dist",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "frameworksBackend": {
      "region": "asia-east1"
    }
  }
}
```

## CICD パイプライン構成

### 全体フロー
1. **Build Phase**: 依存関係インストール → ビルド → アーティファクト保存
2. **Test Phase**: テスト実行
3. **Deploy Phase**: Firebase Hosting へのデプロイ

### トリガー条件
- `main` ブランチへのプッシュ

### 必要なGitHub Secrets
- `FIREBASE_SERVICE_ACCOUNT_JSON`: Firebase サービスアカウントキー（JSON形式またはBase64形式）
- `FIREBASE_PROJECT_ID`: Firebase プロジェクトID

## 修正履歴

### コミット履歴
```
665cd28 - fix pipline (最新)
2514ae1 - fix pipline  
a3a072e - fix pipline & .firebaserc
ff1d970 - delete webframeworks
d94ddea - delete webframeworks
ed38c8a - delete webframeworks
48cc8c0 - JSON 設定、githubCLI経由で
```

### 主要な変更点
1. **サービスアカウント認証の堅牢化**
2. **webframeworks実験機能の削除**
3. **プロジェクト指定の明確化**
4. **検証ステップの追加**

## 学習ポイント

### 1. Firebase認証のベストプラクティス
- サービスアカウントキーの適切な管理
- GitHub Secretsの安全な使用
- 認証情報の検証とデバッグ

### 2. GitHub Actions設計
- アーティファクトの適切な受け渡し
- ジョブ間の依存関係管理
- セキュリティを考慮した秘密情報の扱い

### 3. Firebase Hosting設定
- 適切な設定ファイルの作成
- プロジェクト指定の重要性
- 不要な機能の無効化

## 今後の改善点

1. **エラーハンドリングの強化**
   - より詳細なエラーメッセージ
   - 失敗時の自動リトライ機能

2. **セキュリティの向上**
   - 秘密鍵の適切な削除確認
   - 最小権限の原則に基づくサービスアカウント設定

3. **パフォーマンスの最適化**
   - キャッシュ戦略の改善
   - 並列実行の活用

## 参考資料
- [Firebase CLI Reference](https://firebase.google.com/docs/cli)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Firebase Hosting Guide](https://firebase.google.com/docs/hosting)

---
**作成日**: 2025年8月16日  
**最終更新**: CICDパイプライン成功後
