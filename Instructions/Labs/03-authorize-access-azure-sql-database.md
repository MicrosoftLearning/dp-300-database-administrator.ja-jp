---
lab:
  title: 'ラボ 3: Azure Active Directory を使用して Azure SQL Database へのアクセスを承認する'
  module: Implement a Secure Environment for a Database Service
---

# データベースの認証と承認を構成する

**推定所要時間:20 分**

受講者は、レッスンで得た情報を利用して、Azure portal と *AdventureWorks* データベース内でセキュリティを構成した後、実装します。

あなたは、データベース環境のセキュリティを確保するために、上級データベース管理者として採用されました。

**注:** これらの演習では、T-SQL コードをコピーして貼り付けるように求められます。 コードを実行する前に、コードが正しくコピーされていることを確認してください。

## Azure Active Directory を使用して Azure SQL Database へのアクセスを承認する

1. ラボの仮想マシンからブラウザー セッションを開始し、[https://portal.azure.com](https://portal.azure.com/) に移動します。 このラボ仮想マシンの **[リソース]** タブで提供されている Azure の **[ユーザー名]** と **[パスワード]** を使用してポータルに接続します。

    ![画像 1](../images/dp-300-module-01-lab-01.png)

1. Azure portal のホーム ページで、**[すべてのリソース]** を選択します。

    ![Azure portal ホーム ページのスクリーンショット ([すべてのリソース] を選択)](../images/dp-300-module-03-lab-01.png)

1. Azure SQL Database サーバー **dp300-lab-xxxxxx** (**xxxxxx** はランダムな文字列) を選択し、 **[Active Directory 管理者]** の横の **[未構成]** を選択します。

    ![[未構成] を選択している画面のスクリーンショット](../images/dp-300-module-03-lab-02.png)

1. 次の画面で **[管理者の設定]** を選択します。

    ![[管理者の設定] を選択している画面のスクリーンショット](../images/dp-300-module-03-lab-03.png)

1. **Azure Active Directory** サイド バーで、Azure portal にログインした Azure ユーザー名を検索し、 **[選択]** をクリックします。

1. **[保存]** を選択して手順を完了します。 これで次のように、自分のユーザー名がサーバーの Azure Active Directory 管理者になります。

    ![[Active Directory 管理者] ページのスクリーンショット](../images/dp-300-module-03-lab-04.png)

1. 左側の **[概要]** を選択し、**[サーバー名]** をコピーします。

    ![サーバー名のコピー元を示すスクリーンショット](../images/dp-300-module-03-lab-05.png)

1. SQL Server Management Studio を開いて、**[接続]** > **[データベース エンジン]** を選択します。 **[サーバー名]** に、サーバーの名前を貼り付けます。 認証の種類を **[Azure Active Directory - MFA で汎用]** に変更します。

    ![[サーバーに接続] ダイアログのスクリーンショット](../images/dp-300-module-03-lab-06.png)

    **[ユーザー名]** フィールドでは、 **[リソース]** タブの Azure **[ユーザー名]** を選択します。

1. **[接続]** を選択します。

> [!NOTE]
> Azure SQL データベースへの初回サインイン試行時に、クライアントの IP アドレスをファイアウォールに追加する必要があります。 これは、SQL Server Management Studio によって自動的に行われます。 Azure portal の **[リソース]** タブから入手した**パスワード**を使用して **[サインイン]** を選択し、Azure の資格情報を選択して **[OK]** を選択します。
> ![クライアントの IP アドレスを追加する画面のスクリーンショット](../images/dp-300-module-03-lab-07.png)

## データベース オブジェクトへのアクセスを管理する

このタスクでは、データベースとそのオブジェクトへのアクセスを管理します。 まず、*AdventureWorksLT* データベースに 2 人のユーザーを作成します。

1. **[オブジェクト エクスプローラー]** を使用し、**[データベース]** を展開します。
1. **[AdventureWorksLT]** を右クリックし、**[新しいクエリ]** を選択します。

    ![[新しいクエリ] メニュー オプションのスクリーンショット](../images/dp-300-module-03-lab-08.png)

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

    ![エラー メッセージ "オブジェクト 'DemoProc' で EXECUTE アクセス許可が拒否されました" のスクリーンショット](../images/dp-300-module-03-lab-09.png)

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

    ![ストアド プロシージャから返されたデータ行を示すスクリーンショット](../images/dp-300-module-03-lab-10.png)

この演習では、Azure Active Directory を使用して、Azure でホストされている SQL Server へのアクセス権を Azure の資格情報に付与する方法を確認しました。 また、T-SQL ステートメントを使用して新しいデータベース ユーザーを作成し、ストアド プロシージャを実行するためのアクセス許可を付与しました。
