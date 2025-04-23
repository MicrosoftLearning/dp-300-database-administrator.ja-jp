---
lab:
  title: ラボ 3 – Microsoft Entra ID を使用して Azure SQL Database へのアクセスを承認する
  module: Implement a Secure Environment for a Database Service
---

# データベースの認証と承認を構成する

**推定所要時間: 25 分**

受講者は、レッスンで得た情報を利用して、Azure portal と *AdventureWorksLT* データベース内でセキュリティを構成した後、実装します。

あなたは、データベース環境のセキュリティを確保するために、上級データベース管理者として採用されました。

> &#128221; これらの演習では、T-SQL コードをコピーして貼り付け、既存の SQL リソースを使うように求められます。 コードを実行する前に、コードが正しくコピーされていることを確認してください。

## 環境をセットアップします

ラボ仮想マシンが提供され、事前に構成されている場合は、**C:\LabFiles** フォルダーにラボ ファイルが用意されているはずです。 *少し時間をとって確認してください。ファイルが既に存在存在している場合には、このセクションをスキップしてください*。 ただし、独自のマシンを使用している場合、またはラボ ファイルが見つからない場合は、 *GitHub* からそれらを複製して続行する必要があります。

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、Visual Studio Code のセッションを起動します。

1. コマンド パレット (Ctrl+Shift+P) を開き、「**Git Clone**」と入力します。 **[Git: Clone]** オプションを選択します。

1. **[Repository URL]** フィールドに次の URL を貼り付け、**Enter** キーを押します。

    ```url
    https://github.com/MicrosoftLearning/dp-300-database-administrator.git
    ```

1. リポジトリを、ラボ仮想マシン、または提供されていない場合はローカル コンピューターの **[C:\LabFiles]** フォルダーに保存してください（フォルダーが存在しない場合は作成します）。

## Azure で SQL Server を構成する

Azure にログインし、Azure で実行されている既存の Azure SQL Server インスタンスがあるかどうかを確認します。 *Azure で SQL Server インスタンスが既に実行されている場合は、このセクションをスキップします*。

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、Visual Studio Code セッションを起動し、前のセクションから複製されたリポジトリに移動します。

1. **/Allfiles/Labs** フォルダーを右クリックし、**[統合ターミナルで開く]** を選択します。

1. Azure CLI を使用して Azure に接続しましょう。 次のコマンドを入力して、**Enter** キーを押します。

    ```bash
    az login
    ```

    > &#128221; ブラウザー ウィンドウが開くことに注意してください。 ログインには Azure 資格情報を使用してください。

1. Azure にログインしたら、リソース グループがまだ存在しない場合は作成し、そのリソース グループの下に SQL サーバーとデータベースを作成します。 次のコマンドを入力して、**Enter** キーを押します。 *スクリプトが完了するまで数分かかります*。

    ```bash
    cd ./Setup
    ./deploy-sql-database.ps1
    ```

    > &#128221; 既定では、このスクリプトは **contoso-rg** というリソース グループを作成するか、名前が *contoso-rg* (存在する場合) で始まるリソースを使用していることに注意してください。 既定では、 **West US 2** リージョン (westus2) にもすべてのリソースが作成されます。 最後に、 **SQL 管理者パスワードのランダムな 12 文字のパスワード**を生成します。 これらの値は、パラメーター **-rgName**、**-location**、**-sqlAdminPw** のいずれかまたは複数を使用して、任意の値に変更することができます。 パスワードは Azure SQL のパスワードの複雑さの要件を満たす必要があります。具体的には、12文字以上で、大文字 1 文字、小文字 1 文字、数字 1 つ、特殊文字 1 つを含める必要があります。

    > &#128221; スクリプトによって、現在のパブリック IP アドレスが SQL Server ファイアウォール規則に追加されることに注意してください。

1. スクリプトが完了すると、リソース グループ名、SQL Server 名とデータベース名、管理者ユーザー名とパスワードが返されます。 これらの値は、後でラボで必要になりますのでメモしておきます。

---

## Microsoft Entra ID を使用して Azure SQL Database へのアクセスを承認する

`CREATE USER [anna@contoso.com] FROM EXTERNAL PROVIDER` T-SQL 構文を使って、Microsoft Entra アカウントから包含データベース ユーザーとしてログインを作成できます。 包含データベース ユーザーは、データベースに関連付けられた Microsoft Entra ディレクトリ内の ID にマップされており、`master` データベース内にログインはありません。

Azure SQL Database に Microsoft Entra サーバー ログインを導入することで、SQL Database の仮想 `master` データベース内に Microsoft Entra プリンシパルからログインを作成できます。 Microsoft Entra ログインは、Microsoft Entra の "ユーザー、グループ、サービス プリンシパル" から作成できます。** 詳しくは、「[Microsoft Entra のサーバー プリンシパル](/azure/azure-sql/database/authentication-azure-ad-logins)」をご覧ください

さらに、Azure portal は管理者の作成にのみ使用できます。Azure のロールベースのアクセス制御ロールは、Azure SQL Database 論理サーバーには反映されません。 Transact-SQL (T-SQL) を使用して、追加のサーバーとデータベースのアクセス許可を付与する必要があります。 SQL Server の Microsoft Entra 管理者を作成しましょう。

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、ブラウザー セッションを起動し、 [https://portal.azure.com](https://portal.azure.com/)に移動します。 Azure 資格情報を使用して、Azure portal に接続します。

1. Azure portal ホームページで、**[SQL Server]** を検索して選択します。

1. SQL Server **dp300-lab-xxxxxxxx** を選択します。*xxxxxxxx* はランダムな数値文字列です。

    > &#128221; このラボで作成されていない独自の Azure SQL Server を使用している場合は、その SQL Server の名前を選択します。

1. *[概要]* ブレードで、*[Microsoft Entra 管理者]* の横にある **[未構成]** を選択します。

1. 次の画面で **[管理者の設定]** を選択します。

1. **[Microsoft Entra ID]** サイド バーで、Azure portal にログインした Azure ユーザー名を検索し、 **[選択]** をクリックします。

1. **[保存]** を選択して手順を完了します。 これにより、ユーザー名がサーバーの Microsoft Entra 管理者になります。

1. 左側の **[概要]** を選択し、**[サーバー名]** をコピーします。

1. SQL Server Management Studio (SSMS) を開いて、**[接続]** > **[データベース エンジン]** を選択します。 **[サーバー名]** に、サーバーの名前を貼り付けます。 [認証の種類] を **[Microsoft Entra MFA]** に変更します。

1. **[接続]** を選択します。

## データベース オブジェクトへのアクセスを管理する

このタスクでは、データベースとそのオブジェクトへのアクセスを管理します。 まず、*AdventureWorksLT* データベースに 2 人のユーザーを作成します。

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、SSMS で、Azure Server 管理者アカウントまたは Microsoft Entra 管理者アカウントを使用して *AdventureWorksLT* データベースにログインします。

1. **[オブジェクト エクスプローラー]** を使用し、**[データベース]** を展開します。

1. **[AdventureWorksLT]** を右クリックし、**[新しいクエリ]** を選択します。

1. 次の T-SQL をコピーして、新しいクエリ ウィンドウに貼り付けます。 クエリを実行すると、2 人のユーザーが作成されます。

    ```sql
    CREATE USER [DP300User1] WITH PASSWORD = 'Azur3Pa$$';
    GO

    CREATE USER [DP300User2] WITH PASSWORD = 'Azur3Pa$$';
    GO
    ```

    **注:** これらのユーザーは、AdventureWorksLT データベースのスコープ内で作成されます。 次に、カスタム ロールを作成して、そこにユーザーを追加します。

1. 同じクエリ ウィンドウで次の T-SQL を実行します。

    ```sql
    CREATE ROLE [SalesReader];
    GO

    ALTER ROLE [SalesReader] ADD MEMBER [DP300User1];
    GO

    ALTER ROLE [SalesReader] ADD MEMBER [DP300User2];
    GO
    ```

    次に、**SalesLT** スキーマに新しいストアド プロシージャを作成します。

1. クエリ ウィンドウで次の T-SQL を実行します。

    ```sql
    CREATE OR ALTER PROCEDURE SalesLT.DemoProc
    AS
    SELECT P.Name, Sum(SOD.LineTotal) as TotalSales ,SOH.OrderDate
    FROM SalesLT.Product P
    INNER JOIN SalesLT.SalesOrderDetail SOD on SOD.ProductID = P.ProductID
    INNER JOIN SalesLT.SalesOrderHeader SOH on SOH.SalesOrderID = SOD.SalesOrderID
    GROUP BY P.Name, SOH.OrderDate
    ORDER BY TotalSales DESC
    GO
    ```

    次に、セキュリティをテストするために、`EXECUTE AS USER` 構文を使用します。 これでデータベース エンジンに、ユーザーのコンテキストでクエリを実行させることができます。

1. 次の T-SQL を実行します。

    ```sql
    EXECUTE AS USER = 'DP300User1'
    EXECUTE SalesLT.DemoProc
    ```

    次のエラー メッセージが表示されます。

    <span style="color:red">メッセージ 229、レベル 14、状態 5、プロシージャ SalesLT.DemoProc、1 行目 [バッチ開始行 0] オブジェクト 'DemoProc'、データベース 'AdventureWorksLT'、スキーマ 'SalesLT' に対する EXECUTE アクセス許可が拒否されました。</span>

1. 次に、ストアド プロシージャを実行するためのアクセス許可をロールに付与します。 次の T-SQL を実行します。

    ```sql
    REVERT;
    GRANT EXECUTE ON SCHEMA::SalesLT TO [SalesReader];
    GO
    ```

    1 つ目のコマンドで、実行コンテキストがデータベースの所有者に戻ります。

1. 前の T-SQL を再実行します。

    ```sql
    EXECUTE AS USER = 'DP300User1'
    EXECUTE SalesLT.DemoProc
    ```

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

### LabFiles フォルダーを削除する

このラボ用に新しい LabFiles フォルダーを作成して、それが不要になった場合は、LabFiles フォルダーを削除して、このラボで作成されたすべてのファイルを削除できます。

1. ラボ仮想マシン、または提供されていない場合はローカル コンピューターから、エクスプローラーを開き、 **C:\\** ドライブに移動します。
1. **LabFiles** フォルダーを右クリックし、**[削除]** を選択します。
1. **[はい]** を選択して、フォルダーの削除を確認します。

---

このラボは以上で完了です。

この演習では、Microsoft Entra ID を使用して、Azure でホストされている SQL Server へのアクセス権を Azure の資格情報に付与する方法を見てきました。 また、T-SQL ステートメントを使用して新しいデータベース ユーザーを作成し、ストアド プロシージャを実行するためのアクセス許可を付与しました。
