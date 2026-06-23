---
lab:
  title: 'Lab 17 – ケース スタディ: Azure SQL で小売業のセマンティック検索ワークロードをセキュリティで保護する'
  module: Plan and Secure Azure SQL for AI Workloads
  description: 小売業のセマンティック検索のケース スタディに取り組み、AI 対応データベースに関して DBA が担当する 5 つの運用前タスクを完了します。 最終的には、AI ワークロード向けの Azure SQL を計画し、セキュリティで保護する方法を理解できるようになります。
  duration: 20
  level: 300
  islab: true
  status: in-development
  targetDate: '2026-07-01'
---

# AI ワークロード向けの Azure SQL を計画し、セキュリティで保護する

**推定時間**:20 分間

ある小売企業は、自社のカタログにセマンティック製品検索を追加したいと考えています。 カタログには 800 万個の SKU が含まれています。 最初の埋め込み計画では、単精度の 1,536 次元とし、エラスティック ジョブによって夜間に更新されます。 検索時、アプリケーション トラフィックにより、それらの埋め込みのクエリが実行されます。 

この演習では、すべての DBA がこのようなワークロードに対して担当する 5 つの運用前の決定 (サイズ設定ワークシート、マネージド ID のセットアップ、外部モデルの作成、ロールの割り当て、監査の構成) を行います。 これは、設計のウォークスルーであり、実際にデプロイを行うものではありません。各決定について検討し、それによって生成されたコードを確認しますが、それを完了するために Azure サブスクリプションや稼働中のデータベースは必要ありません。

> [!NOTE]
> この演習の Transact-SQL および Azure CLI のスニペットは **参照例**であり、各決定が実際にどのようなものであるかを示しています。 これらは実行しないでください。 各スニペットに目を通し、それが自身の決定と一致することを確認し、パターンを実際のワークロードに適用する際には、プレースホルダー名 (`sql-retail-prod`、`<openai-resource>` など) を適宜置き換えてください。

| 決定 | それが重要である理由 |
|---|---|
| **ベクター ワークロードのサイズを設定する** | ベクター列とそのインデックスは、一般的なリレーショナル データよりもはるかに多くのストレージを消費します。 事前にリソース要件を見積もっておくと、適切なサービス レベルを選択でき、運用後にコストのかかるサイズ変更やスロットリングを回避できます。 |
| **システム割り当てマネージド ID を有効化する** | マネージド ID による認証は、API キーの保存、ローテーション、漏洩が発生しないことを意味します。 Azure Entra ID によって資格情報が発行され、RBAC によってアクセスが制御されるため、シークレットの取り扱いに関する最も一般的なリスクが排除されます。 |
| **外部モデルを作成する** | 外部モデル オブジェクトは、データベースを特定の Azure OpenAI デプロイと API バージョンにピン留めします。 呼び出し元は 1 つのモデル オブジェクトを参照するため、エンドポイントをすべてのクエリではなく 1 か所で更新できます。 |
| **最小特権ロールを適用する** | 各プリンシパルに必要な特権のみを付与すると、対話型クエリ パスを外部エンドポイントから離すことができます。 これにより、不正なアクセス ルートがブロックされ、埋め込み呼び出しの暴走による予期しないコストの発生が抑制されます。 |
| **監査と Defender を有効にする** | 外部モデルの呼び出しを監査し、Microsoft Defender for SQL を有効にすると、AI 運用に関する検証可能な記録や、脅威に対するプロアクティブなアラートが得られます。これらは、コンプライアンスやインシデント応答に不可欠です。 |

## 決定事項 1: ベクター ワークロードのサイズを設定する

このワークシートに次のシナリオの情報を入力します。

| 入力 | 値 |
|---|---|
| 行数 | 8,000,000 |
| Dimensions | 1,536 |
| 精度 | float32 |
| 行あたりのベクター サイズ | 6 KB |
| ベクター列の合計ストレージ | **?** |
| ベクター指数の推定オーバーヘッド (50% まで) | **?** |
| AI 関連の合計ストレージ | **?** |
| 推奨レベル | **?** |

### Copilot にサイズの見積もりを依頼する

計算を手動で行う代わりに、それを SSMS の GitHub Copilot (または Azure portal の Copilot) に依頼することができます。 Copilot Chat で、既知の入力を含むプロンプトを貼り付けます。

> "Azure SQL でベクター列を計画しています。 8,000,000 行があり、各行には float32 の 1,536 次元の埋め込み (1 次元あたり 4 バイト) が格納されています。 行あたりのベクター サイズ、ベクター列の合計ストレージ、約 50% のベクター指数の推定オーバーヘッド、AI 関連の合計ストレージを推定します。 その後、読み取り負荷の高い夜間の更新と検索のワークロードに適した Azure SQL サービスレベルを推奨します。"

Copilot は、1 行あたりのサイズ、合計、推奨するサービス レベルとその理由を返します。 この結果を出発点とし、以下の計算と照らして数値を検証します。

> [!TIP]
> 常に Copilot の計算と想定を確認してください。 "計算の過程を順を追って示します" (各数値を確認できるようにするため)、"カタログが 5000 万行に増加した場合、推奨されるサービス レベルはどのように変更されますか?" (設計の圧力テストを行うため)  など、フォローアップを依頼します。

計算を行って Copilot の見積もりを検証します。

- **ベクター列の合計数:** 8,000,000 × 6 KB ≈ **48 GB**。
- **列の約 50% のベクター インデックスのオーバーヘッド:** ≈ **24 GB**。
- **AI 関連の合計ストレージ:** ≈ **72 GB**。
- **推奨されるサービス レベル:** **General Purpose、40-vCore** は、夜間の更新と対話型の検索のためのストレージ占有領域や、読み取り負荷の高いクエリ パターンに適しています。 カタログの SKU 数が 12 か月以内に約 5,000 万個を超えると予想される場合は、ストレージの増加とコンピューティングのスケーリングが切り離されるため、代わりに **Hyperscale** を選択します。

## 決定事項 2: マネージド ID を使用して認証する

論理サーバーは、Azure OpenAI で受け入れられる ID が必要です。 この設計は、システム割り当てマネージド ID を有効にし、その id に Azure OpenAI リソースに対する適切な RBAC の役割を付与します。 構成は次のようになります。

```bash
az sql server update \
  --name sql-retail-prod \
  --resource-group rg-retail \
  --identity-type SystemAssigned
```

```bash
SQL_MI_ID=$(az sql server show \
  --name sql-retail-prod \
  --resource-group rg-retail \
  --query identity.principalId -o tsv)

az role assignment create \
  --assignee $SQL_MI_ID \
  --role "Cognitive Services OpenAI User" \
  --scope <azure-openai-resource-id>
```

この構成では、シークレットは作成されておらず、API キーも存在しません。 サーバー ID は、Azure OpenAI への認証パス全体です。

## 決定事項 3: 外部モデル オブジェクトを定義する

データベース内では、3 つのオブジェクトが外部モデルをサポートしています。それらは、マスター キー (まだ存在しない場合)、マネージド ID にバインドされた資格情報、外部モデル オブジェクト自体です。 これらを定義するスクリプトは次のようになります。

```sql
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<strong-password>';

-- The credential name must be the protocol + FQDN of the endpoint the model calls.
CREATE DATABASE SCOPED CREDENTIAL [https://<openai-resource>.openai.azure.com]
WITH IDENTITY = 'Managed Identity',
     SECRET = '{"resourceid": "https://cognitiveservices.azure.com"}';

CREATE EXTERNAL MODEL Ada2Embeddings
WITH (
    LOCATION = 'https://<openai-resource>.openai.azure.com/openai/deployments/text-embedding-ada-002/embeddings?api-version=2023-05-15',
    API_FORMAT = 'Azure OpenAI',
    MODEL_TYPE = EMBEDDINGS,
    MODEL = 'text-embedding-ada-002',
    CREDENTIAL = [https://<openai-resource>.openai.azure.com]
);
```

`LOCATION` 値は、モデルを特定のデプロイと API バージョンに固定します。 デプロイを変更する場合は、すべての呼び出し元のコードではなく、モデル オブジェクトを更新します。

## 決定事項 4: 最小特権ロールの付与を適用する

ロールの設計では、2 つのプリンシパルが必要であり、それぞれに対して必要なロールだけを付与し、それ以上のロールは付与しません。 この付与は、次のようになります。

```sql
-- Refresh service account
CREATE USER [embedding_refresh_svc] WITHOUT LOGIN;
GRANT EXECUTE ON EXTERNAL MODEL::Ada2Embeddings TO [embedding_refresh_svc];
GRANT REFERENCES ON DATABASE SCOPED CREDENTIAL::[https://<openai-resource>.openai.azure.com] TO [embedding_refresh_svc];
GRANT INSERT, UPDATE ON dbo.ProductEmbeddings TO [embedding_refresh_svc];

-- Query user: NO external endpoint permissions
CREATE USER [app_query_user] WITHOUT LOGIN;
GRANT SELECT ON dbo.ProductEmbeddings TO [app_query_user];
```

クエリ ユーザーには、Azure OpenAI へのパスはありません。 検索クエリにより、格納されたベクターが直接読み込まれます。 夜間の更新ジョブだけが外部エンドポイントにアクセスします。

## 決定事項 5: 監査と Defender を有効にする

最後の決定事項では、外部モデルの SQL Audit を有効にし、サーバー上で Microsoft Defender for SQL を有効にすることです。

```sql
CREATE SERVER AUDIT [AuditAIOperations]
TO URL (PATH = 'https://<staudit>.blob.core.windows.net/<container>');

ALTER SERVER AUDIT [AuditAIOperations] WITH (STATE = ON);

CREATE DATABASE AUDIT SPECIFICATION [AIDatabaseAudit]
FOR SERVER AUDIT [AuditAIOperations]
ADD (EXECUTE ON EXTERNAL MODEL::[Ada2Embeddings] BY [public])
WITH (STATE = ON);
```

```bash
az security pricing create --name SqlServers --tier Standard
```

> [!TIP]
> このパターンを実際に適用する際は、セキュリティ ベースラインの完了を宣言する前に、単一の埋め込み生成でエンド ツー エンドのフローを検証してください。 `AI_GENERATE_EMBEDDINGS` が正常に返され、**かつ**監査ログに `embedding_refresh_svc` からの呼び出しがキャプチャされていれば、ベースラインは機能します。 監査行がない場合は、夜間ジョブを実際に実行する前に修正します。

これで、AI 対応データベースに関してすべての DBA が担当する 5 つの運用前の決定 (ワークロードのサイズ設定、ID パスの構成、外部モデルの定義、アクセス許可のスコープ設定、監査の有効化) をすべて完了しました。

## 設計の決定を検証する

小売業のセマンティック検索のケース スタディに基づいて、次の質問に回答します。 各質問の回答を選択し、**[回答を表示]** を展開して、理由を確認します。

**1. 論理サーバーは、システム割り当てマネージド ID を使用して Azure OpenAI に対する認証を行います。この方が API キーをデータベースに保存するよりも好ましいのはなぜですか?**

- **回答。** マネージド ID を使用すると、埋め込みクエリの実行速度が向上する。
- **B**. シークレットの保存、ローテーション、漏洩は発生しない。資格情報の発行とローテーションは Azure Entra ID によって自動的に行われ、アクセスは RBAC によって制御される。
- **C**. Azure SQL Database では API キーはサポートされていない。

<details markdown="1">
<summary>回答を表示</summary>

✅ **正解: B。** シークレットの保存、ローテーション、漏洩は発生しない。資格情報の発行とローテーションは Azure Entra ID によって自動的に行われ、アクセスは RBAC によって制御される。

マネージド ID を使用すると、API キーはデータベースまたはアプリケーションのどこにも存在しません。 サーバー ID は認証パス全体であるため、ソース管理や接続文字列で漏洩する情報はありません。アクセス権の付与や取り消しを行うには、RBAC のロールの割り当てを変更します。 マネージド ID を使用してもクエリのパフォーマンスは変わりません。また、API キーも技術的にはサポートされていますが、セキュリティ態勢が弱くなるだけです。

</details>

**2. ロールの設計では、対話型の検索サービスを提供する `app_query_user` には `dbo.ProductEmbeddings` に対する `SELECT` が付与されますが、外部モデルや資格情報に対するアクセス許可は付与されません。なぜですか?**

- **回答。** 検索クエリは格納されたベクターを直接読み取るため、クエリ ユーザーは外部エンドポイントを呼び出す必要がなく、外部エンドポイントを呼び出すのは夜間の更新ジョブだけである。
- **B**. `SELECT` の付与には、外部モデルへのアクセスが自動的に含まれる。
- **C**. Azure SQL では、クエリ ユーザーに、外部モデルへのアクセス許可を付与することはできない。

<details markdown="1">
<summary>回答を表示</summary>

✅ **正解: A。** 検索クエリは格納されたベクターを直接読み取るため、クエリ ユーザーは外部エンドポイントを呼び出す必要がなく、外部エンドポイントを呼び出すのは夜間の更新ジョブだけである。

これは最小特権の実施です。 埋め込みは `embedding_refresh_svc` によって毎晩 1 回生成され、テーブルに保存されます。 対話型の検索では、単純な `SELECT` を使用して、格納されたベクターとの比較が行われるため、クエリ ユーザーが Azure OpenAI にアクセスするためのパスはありません。 外部エンドポイントへのアクセス許可を単一の更新アカウントに制限すると、攻撃面が縮小され、潜在的なコスト エクスポージャーが削減されます。

</details>

**3. データベース スコープ付き資格情報は `[https://<openai-resource>.openai.azure.com]` という名前です。この名前付けはどんなルールに従っていますか?**

- **回答。** 名前は、DBA が選択する任意の説明的な文字列にすることができる。
- **B**. 名前は、プロトコルと、モデルが呼び出すエンドポイントの完全修飾ドメイン名 (FQDN) を合わせたものでなければならない。
- **C**. 名前は、Azure OpenAI デプロイ名と一致する必要がある。

<details markdown="1">
<summary>回答を表示</summary>

✅ **正解: B。** 名前は、プロトコルと、モデルが呼び出すエンドポイントの完全修飾ドメイン名 (FQDN) を合わせたものでなければならない。

Azure SQL は、資格情報と送信呼び出しを URL で照合するため、資格情報名は、ターゲット エンドポイントのプロトコルと FQDN にする必要があります (例: `https://<openai-resource>.openai.azure.com`)。 特定のデプロイや API バージョンは、資格情報名内ではなく外部モデルの `LOCATION` 内で固定されます。

</details>

**4. カタログの SKU 数は現在 800 万個であり、General Purpose 40-vCore サービス レベルで十分対応できるサイズです。このケース スタディで代わりに Hyperscale が推奨されるのは、どのような条件下ですか?**

- **回答。** データベースでベクトル検索を使用する場合は常時。
- **B**. ワークロードで、すべてのクエリでリアルタイムの埋め込み生成を必要とする場合のみ。
- **C**. カタログの SKU 数が 12 か月以内に約 5,000 万個を超えると予想される場合。Hyperscale の場合、ストレージの増加とコンピューティングのスケーリングが切り離されているためである。

<details markdown="1">
<summary>回答を表示</summary>

✅ **正解: C。** カタログの SKU 数が 12 か月以内に約 5,000 万個を超えると予想される場合。Hyperscale の場合、ストレージの増加とコンピューティングのスケーリングが切り離されているためである。

General Purpose は、現在の 72 GB までの AI 関連のメモリ占有領域と、読み取り負荷の高い夜間の更新と検索のパターンに適しています。 ストレージの急速な増大が予想される場合、ストレージをコンピューティングとは独立してスケーリングできるため、Hyperscale がより適した選択肢になります。 ベクトル検索だけであれば Hyperscale は必要ありませんが、ここでの設計では、すべてのクエリで埋め込みを生成するのではなく、それらを格納します。

</details>
