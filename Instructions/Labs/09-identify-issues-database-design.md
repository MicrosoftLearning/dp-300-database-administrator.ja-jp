---
lab:
  title: 'ラボ 9: データベース設計の問題を特定する'
  module: Optimize query performance in Azure SQL
---

# データベース設計の問題を特定する

**推定所要時間: 15 分**

学生は、レッスンで得た情報を利用して、AdventureWorks 内のデジタルトランスフォーメーション プロジェクトの成果物を調べます。 受講生は、Azure portal と他のツールを調べ、ネイティブ ツールを利用してパフォーマンス関連の問題を特定して解決する方法を決定します。 最後に、受講者は、正規化、データ型の選択、インデックス設計の問題について、データベース設計を評価できるようになります。

あなたは、パフォーマンス関連の問題を見つけ出して、見つかった問題を解決するための実用的ソリューションを提供するデータベース管理者として雇用されています。 AdventureWorks は、10 年以上にわたって自転車と自転車部品を消費者と販売業者に直接販売しています。 あなたの仕事は、このモジュールで学習した手法を使用して、クエリのパフォーマンスの問題を明らかにし、解決することです。

> &#128221; これらの演習では、T-SQL コードをコピーして貼り付けるように求められます。 コードを実行する前に、コードが正しくコピーされていることを確認してください。

## 環境をセットアップします

ラボ仮想マシンが提供され、事前に構成されている場合は、**C:\LabFiles** フォルダーにラボ ファイルが用意されているはずです。 *少し時間をとって確認してください。ファイルが既に存在存在している場合には、このセクションをスキップしてください*。 ただし、独自のマシンを使用している場合、またはラボ ファイルが見つからない場合は、 *GitHub* からそれらを複製して続行する必要があります。

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、Visual Studio Code のセッションを起動します。

1. コマンド パレット (Ctrl+Shift+P) を開き、「**Git Clone**」と入力します。 **[Git: Clone]** オプションを選択します。

1. **[Repository URL]** フィールドに次の URL を貼り付け、**Enter** キーを押します。

    ```url
    https://github.com/MicrosoftLearning/dp-300-database-administrator.git
    ```

1. リポジトリを、ラボ仮想マシン、または提供されていない場合はローカル コンピューターの **[C:\LabFiles]** フォルダーに保存してください（フォルダーが存在しない場合は作成します）。

---

## データベースを復元する

既に **AdventureWorks2017** データベースを復元済みの場合は、このセクションをスキップしてもかまいません。

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、SQL Server Management Studio (SSMS) のセッションを起動します。

1. SSMS が開くと、既定で **[サーバーへ接続]** ダイアログが表示されます。 既定のインスタンスを選択し、**[接続]** を選択します。 場合によっては、 **[Trust Server Certificate] (サーバー証明書を信頼する)** チェック ボックスをオンにする必要があります。

    > &#128221; 独自の SQL Server インスタンスを使用している場合は、適切なサーバー インスタンス名と資格情報を使用して接続する必要があることに注意してください。

1. **Databases** フォルダーを選択し、**[新しいクエリ]** を選択します。

1. 次の T-SQL をコピーして、新しいクエリ ウィンドウに貼り付けます。 クエリを実行してデータベースを復元します。

    ```sql
    RESTORE DATABASE AdventureWorks2017
    FROM DISK = 'C:\LabFiles\dp-300-database-administrator\Allfiles\Labs\Shared\AdventureWorks2017.bak'
    WITH RECOVERY,
          MOVE 'AdventureWorks2017' 
            TO 'C:\LabFiles\AdventureWorks2017.mdf',
          MOVE 'AdventureWorks2017_log'
            TO 'C:\LabFiles\AdventureWorks2017_log.ldf';
    ```

    > &#128221; **C:\LabFiles** という名前のフォルダーが必要です。 このフォルダーがない場合は、作成するか、データベースとバックアップ ファイルの別の場所を指定します。

1. **[メッセージ]** タブに、データベースが正常に復元されたことを示すメッセージが表示されます。

## クエリを調べて問題を識別する

1. **[新しいクエリ]** を選択します。 次の T-SQL コードをコピーして、クエリ ウィンドウに貼り付けます。 **[Execute]** を選択してこのクエリを実行します。

    ```sql
    USE AdventureWorks2017

    GO
    
    SELECT BusinessEntityID, NationalIDNumber, LoginID, HireDate, JobTitle
    FROM HumanResources.Employee
    WHERE NationalIDNumber = 14417807;
    ```

1. クエリを実行する前に、**[実行]** ボタンの右側にある **[実際の実行プランを含める]** アイコンを選択するか、**CTRL + M** キーを押します。 これにより、クエリを実行すると実行プランが表示されるようになります。 **[実行]** を選択してこのクエリを実行します。

1. SSMS の結果パネルで **[実行プラン]** タブを選択して、実行プランに移動します。 ご覧のように、**SELECT**演算子には、感嘆符を含む黄色の三角形があります。 これは、演算子に関連付けられた警告メッセージがあることを示します。 警告アイコンにカーソルを合わせて、メッセージを表示し、警告メッセージを確認します。

    > &#128221; 警告メッセージは、クエリに暗黙的な型変換があることを示します。 つまり、SQL Server クエリ オプティマイザーでは、クエリを実行するために、クエリ内のいずれかの列のデータ型を別のデータ型に変換する必要がありました。

## 警告の問題を解決する方法を特定する

*[HumanResources].[Employee]* テーブルの構造は、次のデータ定義言語 (DDL) ステートメントで定義されます。 前の SQL クエリで使用されているフィールドを、その型に注意しながらこの DDL で確認します。

```sql
CREATE TABLE [HumanResources].[Employee](
     [BusinessEntityID] [int] NOT NULL,
     [NationalIDNumber] [nvarchar](15) NOT NULL,
     [LoginID] [nvarchar](256) NOT NULL,
     [OrganizationNode] [hierarchyid] NULL,
     [OrganizationLevel] AS ([OrganizationNode].[GetLevel]()),
     [JobTitle] [nvarchar](50) NOT NULL,
     [BirthDate] [date] NOT NULL,
     [MaritalStatus] [nchar](1) NOT NULL,
     [Gender] [nchar](1) NOT NULL,
     [HireDate] [date] NOT NULL,
     [SalariedFlag] [dbo].[Flag] NOT NULL,
     [VacationHours] [smallint] NOT NULL,
     [SickLeaveHours] [smallint] NOT NULL,
     [CurrentFlag] [dbo].[Flag] NOT NULL,
     [rowguid] [uniqueidentifier] ROWGUIDCOL NOT NULL,
     [ModifiedDate] [datetime] NOT NULL
) ON [PRIMARY]
```

1. 実行プランに表示された警告メッセージに従って、どのような変更を推奨しますか?

    1. 暗黙的な変換の原因となっているフィールドとその理由を識別します。
    1. クエリを見てみましょう。

        ```sql
        SELECT BusinessEntityID, NationalIDNumber, LoginID, HireDate, JobTitle
        FROM HumanResources.Employee
        WHERE NationalIDNumber = 14417807;
        ```

        **14417807** は、引用符で囲まれた文字列に含まれていないため、**WHERE**句の *NationalIDNumber* 列と比較される値は数値として比較されることに注意してください。

        テーブル構造を調べると、*NationalIDNumber* 列には、**INT**データ型ではなく、**NVARCHAR** データ型が使用されていることがわかります。 この不整合により、データベース オプティマイザーは、数値を *NVARCHAR* に暗黙的に変換します。その結果、最適ではないプランが作成され、クエリのパフォーマンスに追加のオーバーヘッドが発生します。

暗黙的な変換の警告を修正するために実装できる方法が 2 つあります。 次の手順では、これらの各方法を調査します。

### コードを変更する

1. 暗黙の変換を解決するには、コードをどのように変更しますか? コードを変更し、クエリを再実行します。

    まだであれば、 **[実際の実行プランを含める]** をオンにする (**CTRL + M** キー) ことを忘れないでください。 

    このシナリオでは、値の両側に一重引用符を追加するだけで、値が数値から文字形式に変更されます。 このクエリのクエリ ウィンドウを開いたままにしておきます。

    更新された SQL クエリを実行します。

    ```sql
    SELECT BusinessEntityID, NationalIDNumber, LoginID, HireDate, JobTitle
    FROM HumanResources.Employee
    WHERE NationalIDNumber = '14417807';
    ```

    > &#128221; 警告メッセージが消え、クエリ プランが改善されていることに注意してください。 *NationalIDNumber* 列と比較される値が、テーブル内の列のデータ型と一致するように *WHERE* 句を変更すると、オプティマイザーでの暗黙的な変換を取り除き、より最適なプランを生成することができました。

### データ型を変更する

1. テーブル構造を変更することで、暗黙的な変換の警告を修正することもできます。

    インデックスの修正を試みるには、次のクエリをコピーして新しいクエリ ウィンドウに貼り付けて、列のデータ型を変更します。 **[実行]** を選択するか、<kbd>F5</kbd> キーを押して、クエリの実行を試みます。

    ```sql
    ALTER TABLE [HumanResources].[Employee] ALTER COLUMN [NationalIDNumber] INT NOT NULL;
    ```

    *NationalIDNumber* 列のデータ型を INT に変更すると、変換の問題が解決されます。 ただし、この変更により、データベース管理者として解決する必要のある別の問題が発生します。 上記のクエリを実行すると、次のエラー メッセージが表示されます:

    <span style="color:red">Msg 5074、Level 16、Sate 1、Line1  インデックス 'AK_Employee_NationalIDNumber' は、列 'NationalIDNumber’ に依存しています。Msg 4922、Level 16、State 9、Line 1  ALTER TABLE ALTER COLUMN NationalIDNumber は、1 つ以上のオブジェクトがこの列にアクセスしたため失敗しました。</span>

    *NationalIDNumber* 列は既に存在する非クラスター化インデックスの一部であるため、データ型を変更するにはインデックスを再構築または再作成する必要があります。 **これにより、運用環境でのダウンタイムが長くなる可能性があります。これは、設計で適切なデータ型を選択することの重要性を浮き彫りにするものです。**

1. この問題を解決するには、次のコードをコピーしてクエリ ウィンドウに貼り付け、**[実行]** を選択して実行します。

    ```sql
    USE AdventureWorks2017

    GO
    
    --Dropping the index first
    DROP INDEX [AK_Employee_NationalIDNumber] ON [HumanResources].[Employee]

    GO

    --Changing the column data type to resolve the implicit conversion warning
    ALTER TABLE [HumanResources].[Employee] ALTER COLUMN [NationalIDNumber] INT NOT NULL;

    GO

    --Recreating the index
    CREATE UNIQUE NONCLUSTERED INDEX [AK_Employee_NationalIDNumber] ON [HumanResources].[Employee]( [NationalIDNumber] ASC );

    GO
    ```

1. 次のクエリを実行して、データ型が正常に変更されたことを確認します。

    ```sql
    SELECT c.name, t.name
    FROM sys.all_columns c INNER JOIN sys.types t
        ON (c.system_type_id = t.user_type_id)
    WHERE OBJECT_ID('[HumanResources].[Employee]') = c.object_id
        AND c.name = 'NationalIDNumber'
    ```

1. ここで、実行プランを確認してみましょう。 引用符のない元のクエリを再実行します。

    ```sql
    USE AdventureWorks2017
    GO

    SELECT BusinessEntityID, NationalIDNumber, LoginID, HireDate, JobTitle
    FROM HumanResources.Employee
    WHERE NationalIDNumber = 14417807;
    ```

     クエリ プランを調べると、暗黙的な変換の警告が表示されることなく、整数を使用して *NationalIDNumber* でフィルター処理できるようになったことがわかります。 これで、SQL クエリ オプティマイザーで最適なプランを生成して実行できるようになりました。

>&#128221; 列のデータ型を変更すると、暗黙的な型変換の問題を解決できますが、それが常に最適なソリューションであるとは限りません。 この場合、 *NationalIDNumber* 列のデータ型を **INT** データ型に変更すると、その列のインデックスを削除して再作成する必要があり、運用環境でダウンタイムが発生します。 変更を行う前に、列のデータ型を変更することが既存のクエリとインデックスに与える影響を考慮することが重要です。 さらに、 *NationalIDNumber* 列に依存する他のクエリが **NVARCHAR** データ型である可能性があるため、データ型を変更すると、それらのクエリが中断される可能性があります。

---

## クリーンアップ

データベースまたはラボ ファイルを他の目的で使用していない場合は、このラボで作成したオブジェクトをクリーンアップできます。

### C:\LabFiles フォルダーを削除する

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、**エクスプローラー**を開きます。
1. **C:\\** に移動します。
1. **C:\LabFiles** フォルダーを削除します。

## AdventureWorks2017 データベースを削除する

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、SQL Server Management Studio (SSMS) のセッションを起動します。
1. SSMS が開くと、既定で **[サーバーへ接続]** ダイアログが表示されます。 既定のインスタンスを選択し、**[接続]** を選択します。 場合によっては、 **[Trust Server Certificate] (サーバー証明書を信頼する)** チェック ボックスをオンにする必要があります。
1. **オブジェクト エクスプローラー**で、**[データベース]** フォルダーを展開します。
1. **AdventureWorks2017** データベースを右クリックし、**[削除]** を選択します。
1. **[オブジェクトの削除]** ダイアログで、**[既存の接続を閉じる]** チェックボックスをオンにします。
1. **[OK]** を選択します。

---

このラボは以上で完了です。

この演習では、暗黙的なデータ型変換によって発生するクエリの問題を特定する方法、およびクエリの問題を修正してクエリ プランを改善する方法について学習しました。
