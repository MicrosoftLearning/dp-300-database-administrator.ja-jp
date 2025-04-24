---
lab:
  title: ラボ 7 – 断片化の問題を検出して修正する
  module: Monitor and optimize operational resources in Azure SQL
---

# 断片化の問題を検出して修正する

**推定所要時間:20 分**

学生は、レッスンで得た情報を利用して、AdventureWorks 内のデジタルトランスフォーメーション プロジェクトの成果物を調べます。 受講生は、Azure portal と他のツールを調べ、ネイティブ ツールを利用してパフォーマンス関連の問題を特定して解決する方法を決定します。 最後に、受講者はデータベース内の断片化を特定し、適切に解決する手順を学ぶことができます。

あなたは、パフォーマンス関連の問題を見つけ出して、見つかった問題を解決するための実用的ソリューションを提供するデータベース管理者として雇用されています。 AdventureWorks は、10 年以上にわたって自転車と自転車部品を消費者と販売業者に直接販売しています。 最近、同社は、顧客の依頼対応に使用される製品のパフォーマンスの低下に気づきました。 あなたは SQL ツールを使用してパフォーマンスの問題を特定し、それらを解決する方法を提案する必要があります。

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

## インデックスの断片化を調査する

1. **[新しいクエリ]** を選択します。 次の T-SQL コードをコピーして、クエリ ウィンドウに貼り付けます。 **[実行]** を選択してこのクエリを実行します。

    ```sql
    USE AdventureWorks2017

    GO
    
    SELECT i.name Index_Name
     , avg_fragmentation_in_percent
     , db_name(database_id)
     , i.object_id
     , i.index_id
     , index_type_desc
    FROM sys.dm_db_index_physical_stats(db_id('AdventureWorks2017'),object_id('person.address'),NULL,NULL,'DETAILED') ps
     INNER JOIN sys.indexes i ON ps.object_id = i.object_id 
     AND ps.index_id = i.index_id
    WHERE avg_fragmentation_in_percent > 50 -- find indexes where fragmentation is greater than 50%
    ```

    このクエリで、断片化が **50%** を超えるインデックスが報告されるようになります。 クエリから、結果が返されないはずです。

1. インデックスの断片化は、次のようなさまざまな要因によって発生する可能性があります。

    - テーブルまたはインデックスの頻繁な更新。
    - テーブルまたはインデックスへの頻繁な挿入または削除。
    - ページ分割。

    多数の新しいレコードの挿入や削除によって、Person.Address テーブルとそのインデックスの断片化レベルを上昇させるため。 これを行うには、次のクエリを実行します。

    **[New Query]** を選択します。 次の T-SQL コードをコピーして、クエリ ウィンドウに貼り付けます。 **[Execute]** を選択してこのクエリを実行します。

    ```sql
    USE AdventureWorks2017

    GO
    
    -- Insert 60000 records into the Address table    

    INSERT INTO [Person].[Address] 
        ([AddressLine1], [AddressLine2], [City], [StateProvinceID], [PostalCode], [SpatialLocation], [rowguid], [ModifiedDate])
    SELECT 
        'Split Avenue ' + CAST(v1.number AS VARCHAR(10)), 
        'Apt ' + CAST(v2.number AS VARCHAR(10)), 
        'PageSplitTown', 
        100 + (v1.number % 60),  -- 60 different StateProvinceIDs (100-159)
        '88' + RIGHT('000' + CAST(v2.number AS VARCHAR(3)), 3), -- Structured postal codes
        NULL, 
        NEWID(), -- Ensure unique rowguid
        GETDATE()
    FROM master.dbo.spt_values v1
    CROSS JOIN master.dbo.spt_values v2
    WHERE v1.type = 'P' AND v1.number BETWEEN 1 AND 300 
    AND v2.type = 'P' AND v2.number BETWEEN 1 AND 200;
    GO
    
    -- DELETE 25000 records from the Address table
    DELETE FROM [Person].[Address] WHERE AddressID BETWEEN 35001 AND 60000;

    GO

    -- Insert 40000 records into the Address table
    INSERT INTO [Person].[Address] 
        ([AddressLine1], [AddressLine2], [City], [StateProvinceID], [PostalCode], [SpatialLocation], [rowguid], [ModifiedDate])
    SELECT 
        'Fragmented Street ' + CAST(v1.number AS VARCHAR(10)), 
        'Suite ' + CAST(v2.number AS VARCHAR(10)), 
        'FragmentCity', 
        100 + (v1.number % 60),  -- 60 different StateProvinceIDs (100-159)
        '99' + RIGHT('000' + CAST(v2.number AS VARCHAR(3)), 3), -- Structured postal codes
        NULL, 
        NEWID(), -- Ensure a unique rowguid per row
        GETDATE()
    FROM master.dbo.spt_values v1
    CROSS JOIN master.dbo.spt_values v2
    WHERE v1.type = 'P' AND v1.number BETWEEN 1 AND 200 
    AND v2.type = 'P' AND v2.number BETWEEN 1 AND 200;

    GO
    ```

    このクエリでは、多数の新しいレコードの追加や削除によって、Person.Address テーブルとそのインデックスの断片化レベルが上昇します。

1. 最初のクエリをもう一度実行します。 4 つの非常に断片化されたインデックスが表示されるようになるはずです。

1. **[新しいクエリ]** を選択し、次の T-SQL コードをコピーして、新しいクエリ ウィンドウに貼り付けます。 **[Execute]** を選択してこのクエリを実行します。

    ```sql
    SET STATISTICS IO,TIME ON

    GO
        
    USE AdventureWorks2017

    GO
        
    SELECT DISTINCT (StateProvinceID)
        ,count(StateProvinceID) AS CustomerCount
    FROM person.Address
    GROUP BY StateProvinceID
    ORDER BY count(StateProvinceID) DESC;
        
    GO
    ```

    SQL Server Management Studio の結果ペインで、**[メッセージ]** タブを選択します。 **アドレス** テーブルで、クエリによって実行された論理読み取りの数をメモしておきます。

## 断片化されたインデックスを再構築する

1. **[新しいクエリ]** を選択し、次の T-SQL コードをコピーして、新しいクエリ ウィンドウに貼り付けます。 **[Execute]** を選択してこのクエリを実行します。

    ```sql
    USE AdventureWorks2017

    GO
    
    ALTER INDEX [IX_Address_StateProvinceID] ON [Person].[Address] REBUILD PARTITION = ALL 
    WITH (PAD_INDEX = OFF, 
        STATISTICS_NORECOMPUTE = OFF, 
        SORT_IN_TEMPDB = OFF, 
        IGNORE_DUP_KEY = OFF, 
        ONLINE = OFF, 
        ALLOW_ROW_LOCKS = ON, 
        ALLOW_PAGE_LOCKS = ON)
    ```

1. **[新しいクエリ]** を選択し、次のクエリを実行して、**IX_Address_StateProvinceID** インデックスの断片化が 50% を超えなくなったことを確認します。

    ```sql
    USE AdventureWorks2017

    GO
        
    SELECT DISTINCT i.name Index_Name
        , avg_fragmentation_in_percent
        , db_name(database_id)
        , i.object_id
        , i.index_id
        , index_type_desc
    FROM sys.dm_db_index_physical_stats(db_id('AdventureWorks2017'),object_id('person.address'),NULL,NULL,'DETAILED') ps
        INNER JOIN sys.indexes i ON (ps.object_id = i.object_id AND ps.index_id = i.index_id)
    WHERE i.name = 'IX_Address_StateProvinceID'
    ```

    結果を比較すると、**IX_Address_StateProvinceI** の断片化が 88% から 0 に減少したことがわかります。

1. 前のセクションの select ステートメントを再実行します。 Management Studio の**結果**ペインで、**[メッセージ]** タブの論理読み取りをメモしておきます。 *アドレス テーブルのインデックスを再構築する前に検出された論理読み取りの数から変化していますか*?

    ```sql
    SET STATISTICS IO,TIME ON

    GO
        
    USE AdventureWorks2017

    GO
        
    SELECT DISTINCT (StateProvinceID)
        ,count(StateProvinceID) AS CustomerCount
    FROM person.Address
    GROUP BY StateProvinceID
    ORDER BY count(StateProvinceID) DESC;
        
    GO
    ```

インデックスは再構築されているため、可能な限り効率的になり、論理読み取りは減少しているはずです。 これで、インデックスのメンテナンスがクエリのパフォーマンスに影響を与える可能性があることがわかりました。

---

## クリーンアップ

データベースまたはラボ ファイルを他の目的で使用していない場合は、このラボで作成したオブジェクトをクリーンアップできます。

### C:\LabFiles フォルダーを削除する

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、**エクスプローラー**を開きます。
1. **C:\\** に移動します。
1. **C:\LabFiles** フォルダーを削除します。

### AdventureWorks2017 データベースを削除する

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、SQL Server Management Studio (SSMS) のセッションを起動します。
1. SSMS が開くと、既定で **[サーバーへ接続]** ダイアログが表示されます。 既定のインスタンスを選択し、**[接続]** を選択します。 場合によっては、 **[Trust Server Certificate] (サーバー証明書を信頼する)** チェック ボックスをオンにする必要があります。
1. **オブジェクト エクスプローラー**で、**[データベース]** フォルダーを展開します。
1. **AdventureWorks2017** データベースを右クリックし、**[削除]** を選択します。
1. **[オブジェクトの削除]** ダイアログで、**[既存の接続を閉じる]** チェックボックスをオンにします。
1. **[OK]** を選択します。

---

このラボは以上で完了です。

この演習では、インデックスを再構築し、論理読み取りを分析してクエリのパフォーマンスを向上させる方法を学習しました。
