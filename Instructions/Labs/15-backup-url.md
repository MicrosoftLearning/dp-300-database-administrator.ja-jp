---
lab:
  title: ラボ 15 – URL へのバックアップと URL からの復元
  module: Plan and implement a high availability and disaster recovery solution
---

# Backup to URL

**推定所要時間:30 分**

AdventureWorks の DBA は、Azure で URL にデータベースをバックアップし、人為的なエラーが発生した後、それを Azure BLOB ストレージから復元する必要があります。

## 環境をセットアップします

ラボ仮想マシンが提供され、事前に構成されている場合は、**C:\LabFiles** フォルダーにラボ ファイルが用意されているはずです。 *少し時間をとって確認してください。ファイルが既に存在存在している場合には、このセクションをスキップしてください*。 ただし、独自のマシンを使用している場合、またはラボ ファイルが見つからない場合は、 *GitHub* からそれらを複製して続行する必要があります。

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、Visual Studio Code のセッションを起動します。

1. コマンド パレット (Ctrl+Shift+P) を開き、「**Git Clone**」と入力します。 **[Git: Clone]** オプションを選択します。

1. **[Repository URL]** フィールドに次の URL を貼り付け、**Enter** キーを押します。

    ```url
    https://github.com/MicrosoftLearning/dp-300-database-administrator.git
    ```

1. リポジトリを、ラボ仮想マシン、または提供されていない場合はローカル コンピューターの **[C:\LabFiles]** フォルダーに保存してください（フォルダーが存在しない場合は作成します）。

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

## URL へのバックアップを構成する

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、Visual Studio Code のセッションを起動します。

1. 複製されたリポジトリを **C:\LabFiles\dp-300-database-administrator** で開きます。

1. **Allfiles** フォルダーを右クリックし、**[統合ターミナルで開く]** を選択します。 これにより、正しい場所にターミナル ウィンドウが開きます。

1. ターミナルで、次のコマンドを入力し、**Enter キー**を押します。

    ```bash
    az login
    ```

1. ブラウザーを開いて、コードの入力を求めるプロンプトが表示されます。 手順に従って Azure アカウントにログインします。

1. *リソース グループが既にある場合は、この手順をスキップします*。 リソース グループがない場合は、ターミナルで次のコマンドを実行して作成します。 *contoso-rgXXX######* を、リソース グループの一意の名前に置き換えます。 この名前は Azure 全体で一意である必要があります。 場所 (-l) をリソース グループの場所に置き換えます。

    ```bash
    az group create -n "contoso-rglod#######" -l eastus2
    ```

    **######** をランダムな文字列に置き換えます。

1. ターミナルで、次のように入力し、**Enter** キーを押してストレージ アカウントを作成します。 ストレージ アカウントは必ず一意の名前にしてください。 *名前の長さは 3 ～ 24 文字で、数字と小文字のみを使用する必要があります*。 *########* を 8 文字のランダムな文字列に置き換えます。 この名前は Azure 全体で一意である必要があります。 contoso-rgXXX###### を、リソース グループの名前に置き換えます。 最後に、場所 (-l) をリソース グループの場所に置き換えます。

    ```bash
    az storage account create -n "dp300bckupstrg########" -g "contoso-rgXXX########" --kind StorageV2 -l eastus2
    ```

1. 次に、このストレージ アカウントのキーを取得します。これは、以降の手順で使用します。 ストレージ アカウントの一意の名前とリソース グループを使用して、ターミナルで次のコードを実行します。

    ```bash
    az storage account keys list -g contoso-rgXXX######## -n dp300bckupstrg########
    ```

    アカウント キーは、上のコマンドの結果に含まれます。 必ず、前のコマンドで使用したものと同じ名前 (**-n** の後) とリソース グループ (**-g** の後) を使用してください。 **key1** の戻り値を (二重引用符なしで) コピーします。

1. SQL Server 内のデータベースの URL へのバックアップには、ストレージ アカウント内のコンテナーが使用されます。 この手順で、バックアップ ストレージ専用のコンテナーを作成します。 これを行うには、次のコマンドを実行します。

    ```bash
    az storage container create --name "backups" --account-name "dp300bckupstrg########" --account-key "storage_key" --fail-on-exist
    ```

    ここで **dp300bckupstrg########** はストレージ アカウントの作成時に使用された一意のストレージ アカウント名であり、**storage_key** は以前に生成されたキーです。 出力は **true** を返します。

1. コンテナーのバックアップが適切に作成されたことを確認するには、次を実行します。

    ```bash
    az storage container list --account-name "dp300bckupstrg########" --account-key "storage_key"
    ```

    ここで **dp300bckupstrg########** はストレージ アカウントの作成時に使用された一意のストレージ アカウント名であり、**storage_key** は生成されたキーです。

1. コンテナー レベルでの Shared Access Signature (SAS) がセキュリティのために必要です。 ターミナルで、次のコマンドを実行します。

    ```bash
    az storage container generate-sas -n "backups" --account-name "dp300bckupstrg########" --account-key "storage_key" --permissions "rwdl" --expiry "date_in_the_future" -o tsv
    ```

    ここで **dp300bckupstrg########** はストレージ アカウントの作成時に使用された一意のストレージ アカウント名、**storage_key** は生成されたキー、**date_in_the_future** は現在より後の時間です。 **date_in_the_future** は UTC である必要があります。 たとえば、**2025-12-31T00:00Z** にします。これは、2025 年 12 月 31 日の午前 0 時に期限切れになるということです。

    出力は、次のような内容を返却する必要があります。 共有アクセス署名全体をコピーし、**メモ帳**に貼り付けてください。次のタスクで使用します。

    *se=2020-12-31T00%3A00Z&sp=rwdl&sv=2018-11-09&sr=c>c>rnoGlveGql7ILhziyKYUPBq5ltGc/pgzOCNX5rrLdRQ%3D*

## 資格情報の作成

機能が構成されたので、Azure ストレージ アカウントでバックアップ ファイルを BLOB として生成できます。

1. **SQL Server Management Studio (SSMS)** を起動します。

1. SQL Server に接続するように求められます。 **[Windows 認証]** が選択されていることを確認して、 **[接続]** を選択します。

1. **[新しいクエリ]** を選択します。

1. 次の Transact-SQL を使用して、クラウド内のストレージにアクセスするために使用される資格情報を作成します。 適切な値を入力し、 **[実行]** を選択します。

    ```sql
    IF NOT EXISTS  
    (SELECT * 
        FROM sys.credentials  
        WHERE name = 'https://<storage_account_name>.blob.core.windows.net/backups')  
    BEGIN
        CREATE CREDENTIAL [https://<storage_account_name>.blob.core.windows.net/backups]
        WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
        SECRET = '<key_value>'
    END;
    GO  
    ```

    ここで、**<storage_account_name>** の両方の出現箇所は作成された一意のストレージ アカウント名であり、**<key_value>** は次と同じように前のタスクの最後で生成された値です。

    *se=2020-12-31T00%3A00Z&sp=rwdl&sv=2018-11-09&sr=c>c>rnoGlveGql7ILhziyKYUPBq5ltGc/pgzOCNX5rrLdRQ%3D*

1. 資格情報が正常に作成されたかどうかを確認するには、SSMS のオブジェクト エクスプローラーの **[セキュリティ] -> [資格情報]** に移動します。

1. 間違って入力したために資格情報を再作成する必要がある場合は、次のコマンドを使用して削除できます。必ずストレージ アカウントの名前を変更してください。

    ```sql
    -- Only run this command if you need to go back and recreate the credential! 
    DROP CREDENTIAL [https://<storage_account_name>.blob.core.windows.net/backups]  
    ```

## URL にデータベースをバックアップする

1. SSMS を使用して、Transact-SQL で次のコマンドを使い、データベース **AdventureWorks2017** を Azure にバックアップします。

    ```sql
    BACKUP DATABASE AdventureWorks2017   
    TO URL = 'https://<storage_account_name>.blob.core.windows.net/backups/AdventureWorks2017.bak';
    GO 
    ```

    ここで **<storage_account_name>** は、作成された一意のストレージ アカウント名です。 

    エラーが発生した場合は、資格情報の作成時に何も誤入力しなかったことと、すべてが正常に作成されたことを確認してください。

## Azure CLI を使用してバックアップを検証する

ファイルが実際に Azure にあることを確認するには、Storage Explorer (プレビュー) または Azure Cloud Shell を使用します。

1. Visual Studio Code ターミナルに戻り、次の Azure CLI コマンドを実行します。

    ```bash
    az storage blob list -c "backups" --account-name "dp300bckupstrg########" --account-key "storage_key" --output table
    ```

    前のコマンドで使用したのと同じ一意のストレージ アカウント名 ( **--account-name** の後) とアカウント キー ( **--account-key** の後) を必ず使用してください。

    バックアップ ファイルが正常に生成されたことが確認できます。

## ストレージ ブラウザーを使用してバックアップを検証する

1. ブラウザー ウィンドウで、Azure portal に移動し、 **[ストレージ アカウント]** を検索して選択します。

1. バックアップ用に作成した一意のストレージ アカウント名を選択します。

1. 左側のナビゲーションで、**[ストレージ ブラウザー]** を選択します。 **[BLOB コンテナー]** を展開します。

1. **[backups]** を選択します。

1. バックアップ ファイルがコンテナーに保存されていることに注意してください。

## URL から復元

このタスクでは、Azure BLOB ストレージからデータベースを復元する方法について説明します。

1. **SQL Server Management Studio (SSMS)** から **[新しいクエリ]** を選択し、次のクエリを貼り付けて実行します。

    ```sql
    USE AdventureWorks2017;
    GO
    SELECT * FROM Person.Address WHERE AddressId = 1;
    GO
    ```

1. こちらのコマンドを実行して、その顧客の住所を変更します。

    ```sql
    UPDATE Person.Address
    SET AddressLine1 = 'This is a human error'
    WHERE AddressId = 1;
    GO
    ```

1. **手順 1** を再実行して、アドレスが変更されたことを確認します。 ここで、誰かが WHERE 句を使用せずに、または間違った WHERE 句を使用して、数千または数百万件の行を変更した場合を想定してください。 解決策の 1 つは、利用可能な最新のバックアップからデータベースを復元することです。

1. データベースを復元して、顧客名が誤って変更される前の状態に戻すには、次を実行します。

    > &#128221; **SET SINGLE_USER WITH ROLLBACK IMMEDIATE** 構文を使用すると、未処理のトランザクションはすべてロールバックされます。 これにより、アクティブな接続が原因で復元が失敗するのを防ぐことができます。

    ```sql
    USE [master]
    GO

    ALTER DATABASE AdventureWorks2017 SET SINGLE_USER WITH ROLLBACK IMMEDIATE
    GO

    RESTORE DATABASE AdventureWorks2017 
    FROM URL = 'https://<storage_account_name>.blob.core.windows.net/backups/AdventureWorks2017.bak'
    GO

    ALTER DATABASE AdventureWorks2017 SET MULTI_USER
    GO
    ```

    ここで **<storage_account_name>** は、作成した一意のストレージ アカウント名です。

1. **手順 1** を再実行して、顧客名が復元されたことを確認します。

Azure Blob Storage サービスへのバックアップまたはそこからの復元を実行するには、コンポーネントおよび対話式操作を理解しておくことが重要です。

---

## リソースをクリーンアップする

Azure SQL Server を他の目的で使用していない場合は、このラボで作成したリソースをクリーンアップできます。

### リソース グループを削除します

このラボの新しいリソース グループを作成した場合は、リソース グループを削除して、このラボで作成されたすべてのリソースを削除できます。

1. Azure portal で、左側のナビゲーション ペインから **リソース グループ** を選択するか、検索バーで **リソース グループ**を検索して、結果からそのリソース グループを選択します。

1. このラボ用に作成したリソース グループに移動します。 リソース グループには、Azure SQL Server と、このラボで作成されたその他のリソースが含まれます。

1. トップ メニューから **[リソース グループの削除]** を選択します。

1. **[リソース グループの削除]** ダイアログで、リソース グループの名前を入力して確認した後、**[削除]** を選択します。

1. リソース グループが削除されるのを待ちます。

1. Azure Portal を閉じます。

### ラボ リソースのみを削除する

このラボ用の新しいリソース グループを作成せず、リソース グループとその以前のリソースをそのまま残したい場合は、このラボで作成したリソースを削除できます。

1. Azure portal で、左側のナビゲーション ペインから **リソース グループ** を選択するか、検索バーで **リソース グループ**を検索して、結果からそのリソース グループを選択します。

1. このラボ用に作成したリソース グループに移動します。 リソース グループには、Azure SQL Server と、このラボで作成されたその他のリソースが含まれます。

1. ラボで以前に指定した SQL Server 名のプレフィックスが付いたすべてのリソースを選択します。

1. トップ メニューから、**削除**を選択します。

1. **[リソースの削除]** ダイアログで、「**delete**」と入力し、**[削除]** を選択します。

1. **[削除]** をもう一度選択し、リソースの削除を確定します。

1. リソースが削除されるまで待ちます。

1. Azure Portal を閉じます。

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

これで、Azure で URL にデータベースをバックアップし、必要に応じて復元できることがわかりました。
