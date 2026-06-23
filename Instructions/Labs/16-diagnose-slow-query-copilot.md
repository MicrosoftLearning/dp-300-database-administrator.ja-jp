---
lab:
  title: ラボ 16 – Copilot を使用した低速クエリの診断
  module: Optimize query performance in Azure SQL
  description: SSMS の GitHub Copilot を使って、低速のストアド プロシージャを調査し、インデックス推奨事項を作成し、パフォーマンス ベースラインと照らして検証し、修正を適用します。 最終的には、検証ゲートに従って低速クエリを Copilot で診断し解決する方法が理解できるようになります。
  duration: 20
  level: 300
  islab: true
  status: released
  targetDate: '2026-06-18'
---

# Copilot で低速クエリを診断する

**推定時間**:20 分間

あなたは **ContosoOps** データベースの DBA です。 オペレーション チームは `dbo.usp_GetOpenWorkOrdersByTechnician` の動作が遅く、ミリ秒で返されるはずのクエリが数秒間かかっていると報告しています。 ジョブ: SSMS で GitHub Copilot を使用して、前のユニットの検証ゲートに従って、調査、修正プログラムの生成、検証、適用を行います。

## 前提条件

- **AI 支援**ワークロード (Visual Studio Installer で追加) を含めて SSMS がインストールされていること
- アクティブな GitHub Copilot サブスクリプション
- **ContosoOps** データベースを作成できる SQL Server インスタンスまたは Azure SQL Database

> [!NOTE]
> また、低速でスキャン負荷の高いクエリを含むストアド プロシージャが含まれているデータベースを使い、手順を進めることもできます。 ステップや検証パターンはスキーマに関わらず同一です。

---

## 環境をセットアップします

セットアップ スクリプトは、このラボを通して使用する **ContosoOps** データベースを作成およびシードします。 ラボ ファイルが既に **C:\LabFiles** フォルダーにある場合は、複製をスキップできます。それ以外の場合は、最初に **C:\LabFiles ** このリポジトリを複製します。

1. SSMS を開き、SQL Server インスタンスまたは Azure SQL Database に接続します。

1. **C:\LabFiles\dp-300-database-administrator\Instructions\Templates\01-ContosoOps-Setup.sql** ファイルを開き、**[実行]** を選択します。 このスクリプトは次の内容を行います。

    - **ContosoOps**データベース、`Technicians` および `WorkOrders` テーブル、`dbo.usp_GetOpenWorkOrdersByTechnician` ストアド プロシージャを作成します。
    - サポートするインデックスがない状態で、約 200 万の `WorkOrders` 行をシードするため、ストアド プロシージャがテーブルをスキャンし実行速度が遅くなります。

    > &#128221; シードの手順では、約 200 万行が挿入され、完了するまでに数分かかります。 **セットアップ完了**のメッセージが表示されるまで待ってから続行します。

---

## SSMS で低速なクエリを開き、GitHub Copilot で解析する

1. SSMS で、**ContosoOps** データベースに対して新しいクエリ ウィンドウを開きます。

1. **[実際の実行プランを含める]** を有効にし (または **Ctrl + M** を押し)、I/O と時間統計を有効にしてストアド プロシージャを実行し、低速であることを確認します。

   ```sql
   SET STATISTICS IO ON;
   SET STATISTICS TIME ON;
   GO

   EXEC dbo.usp_GetOpenWorkOrdersByTechnician @TechnicianID = 42;
   GO

   SET STATISTICS IO OFF;
   SET STATISTICS TIME OFF;
   ```

   メッセージ タブの **[論理的読み取り数]** と **[経過時間]** を記録しておきます。これがベースラインです。 実行プランでは、`WorkOrders` の **テーブルスキャン**(クラスター化インデックス スキャン) を確認します。

1. ストアド プロシージャの定義を開きます。 **[オブジェクト エクスプローラー]** で **[ContosoOps]** > **[プログラミング]** > **[ストアド プロシージャ]** を展開し、`dbo.usp_GetOpenWorkOrdersByTechnician` を右クリックし、**[変更]** を選択するとクエリ テキストが表示されます。

1. プロシージャ内の `SELECT` ステートメントを選択し、選択部分を右クリックして **[Copilot で説明]** を選択します。 GitHub Copilot のチャット パネルが開き、クエリが説明されます。 この説明では、Copilot は、サポート インデックスがない `TechnicianID` と `Status` に対するフィルター述語によって発生する `WorkOrders` のテーブル スキャンを特定します。

---

## インデックスの推奨事項を Copilot に依頼する

1. Copilot Chat で、次のように入力します。

   > "このクエリには、TechnicianID と Status でフィルター処理された WorkOrders のテーブル スキャンがあります。 どのようなインデックスを勧めますか?"

1. Copilot は以下のようなインデックスの推奨事項を返します。

    ```sql
    -- Copilot-suggested index
    CREATE NONCLUSTERED INDEX IX_WorkOrders_TechnicianID_Status
    ON dbo.WorkOrders (TechnicianID, Status)
    INCLUDE (WorkOrderID, OpenedDate, Description);
    ```

1. Copilot がインデックスと共に提供する説明を読みます。 キー列の順序が次のように説明されます。最初に等値述語として `TechnicianID`、2 番目に範囲または等値述語として `Status`、次に `INCLUDE` 列がキー参照を回避する理由が説明されます。

> [!NOTE]
> Copilot が提案する `INCLUDE` 列は、それがクエリ テキストから推論できる列に依存します。 ストアド プロシージャで、Copilot のコンテキストにない追加の列を選択する場合は、インデックスを作成する前に、それらを `INCLUDE` リストに追加します。

---

## 推奨事項を検証する

インデックスを作成する前に、私たちが学んだ検証ゲートを適用します。

1. 冗長なインデックスを回避するため、同様のインデックスが既に存在するかどうかを確認します。

    ```sql
    -- Check for existing indexes on WorkOrders
    SELECT
        i.name             AS index_name,
        i.type_desc,
        c.name             AS column_name,
        ic.is_included_column,
        ic.key_ordinal
    FROM sys.indexes AS i
    JOIN sys.index_columns AS ic ON i.object_id = ic.object_id AND i.index_id = ic.index_id
    JOIN sys.columns       AS c  ON ic.object_id = c.object_id AND ic.column_id = c.column_id
    WHERE i.object_id = OBJECT_ID('dbo.WorkOrders')
    ORDER BY i.index_id, ic.key_ordinal;
    ```

    `TechnicianID` のインデックスが既に存在する場合は、`Status` および `INCLUDE` 列が含まれているかどうかを確認します。 含まれている場合は、新しいインデックスは必要ありません。Copilot に既存のインデックスに基づいて推奨事項を絞り込むよう依頼します。

> [!NOTE]
> プロシージャが遅いことを確認したときに、ベースライン (実行プランの論理読み取り、経過時間、テーブル スキャン) を既に記録してあります。 これらの数字をすぐに確認できるようにしておき、インデックス作成後にこれらを比較します。

---

## 適用して確認する

1. Copilot が提案したインデックスを作成します。

    ```sql
    CREATE NONCLUSTERED INDEX IX_WorkOrders_TechnicianID_Status
    ON dbo.WorkOrders (TechnicianID, Status)
    INCLUDE (WorkOrderID, OpenedDate, Description);
    ```

1. ストアド プロシージャを `SET STATISTICS IO ON; SET STATISTICS TIME ON;` でもう一度実行し、新しい論理読み取りと経過時間を、先ほど記録したベースラインと比較します。 論理読み取りが急激に減少し、経過時間が 1 秒未満に低下したことを確認します。

1. 実際の実行プランをもう一度確認します。 プランで `IX_WorkOrders_TechnicianID_Status` に、テーブル スキャンではなく、**Index Seek** が表示されていることを確認します。

1. 実際の環境では、最初に非運用での変更を検証してから、メンテナンス期間中に運用環境のインデックス作成をスケジュールします。

---

## 予想される結果

対象の非クラスター化インデックスでは、`WorkOrders` でのテーブル スキャンが不要になり、ストアド プロシージャは数秒ではなく 1 秒未満で返されます。 完全な検証ゲートに従いました。低速クエリを確認し、ベースラインを記録し、冗長インデックスを確認し、Copilot で推奨されたインデックスを適用して、統計と実行プランで改善を確認しました。

> [!IMPORTANT]
> インデックスの作成自体により、Standard エディションでテーブルが短時間ロックされます。 運用環境の大規模なテーブルの場合は、サービス レベルでサポートされていれば `ONLINE = ON` オプションを使用し、トラフィックの少ない時間帯に操作をスケジュールします。
