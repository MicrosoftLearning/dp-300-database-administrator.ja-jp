---
lab:
  title: 'ラボ 11: Azure Resource Manager テンプレートを使用して Azure SQL Database をデプロイする'
  module: Automate database tasks for Azure SQL
---

# <a name="deploy-an-azure-sql-database-from-a-template"></a>テンプレートから Azure SQL Database をデプロイする

**推定所要時間: 15 分**

あなたはデータベース管理の日常業務を自動化するために、シニア データ エンジニアとして採用されています。 この自動化はピーク時のパフォーマンスで AdventureWorks のデータベースの稼働を確実に続けるうえで役に立つだけでなく、特定の条件に基づいてアラートを生成することもできるようにします。 AdventureWorks では、サービスとしてのインフラストラクチャ (IaaS) とサービスとしてのプラットフォーム (PaaS) の両方のオファリングで SQL Server が使用されています。

## <a name="explore-azure-resource-manager-template"></a>Azure Resource Manager テンプレートを探索する

1. Microsoft Edge で新しいタブを開き、GitHub リポジトリ内の次のパスに移動します。ここには、SQL Database リソースをデプロイするための ARM テンプレートが含まれています

    ```
    https://github.com/Azure/azure-quickstart-templates/tree/master/quickstarts/microsoft.sql/sql-database
    ```

1. **azuredeploy.json** を右クリックし、 **[新しいタブでリンクを開く]** を選択すると、次のような ARM テンプレートが表示されます。

    ```JSON
    {
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "serverName": {
        "type": "string",
        "defaultValue": "[uniqueString('sql', resourceGroup().id)]",
        "metadata": {
            "description": "The name of the SQL logical server."
        }
        },
        "sqlDBName": {
        "type": "string",
        "defaultValue": "SampleDB",
        "metadata": {
            "description": "The name of the SQL Database."
        }
        },
        "location": {
        "type": "string",
        "defaultValue": "[resourceGroup().location]",
        "metadata": {
            "description": "Location for all resources."
        }
        },
        "administratorLogin": {
        "type": "string",
        "metadata": {
            "description": "The administrator username of the SQL logical server."
        }
        },
        "administratorLoginPassword": {
        "type": "securestring",
        "metadata": {
            "description": "The administrator password of the SQL logical server."
        }
        }
    },
    "variables": {},
    "resources": [
        {
        "type": "Microsoft.Sql/servers",
        "apiVersion": "2020-02-02-preview",
        "name": "[parameters('serverName')]",
        "location": "[parameters('location')]",
        "properties": {
            "administratorLogin": "[parameters('administratorLogin')]",
            "administratorLoginPassword": "[parameters('administratorLoginPassword')]"
        },
        "resources": [
            {
            "type": "databases",
            "apiVersion": "2020-08-01-preview",
            "name": "[parameters('sqlDBName')]",
            "location": "[parameters('location')]",
            "sku": {
                "name": "Standard",
                "tier": "Standard"
            },
            "dependsOn": [
                "[resourceId('Microsoft.Sql/servers', concat(parameters('serverName')))]"
            ]
            }
        ]
        }
    ]
    }
    ```

1. JSON プロパティを確認して観察します。

1. **[azuredeploy.json]** タブを閉じ、**sql-database** GitHub フォルダーを含むタブに戻ります。 下にスクロールし、 **[Azure にデプロイ]** を選択します。

    ![[Azure にデプロイ] ボタン](../images/dp-300-module-11-lab-01.png)

1. Azure portal で **[SQL Server とデータベースの作成]** クイックスタート テンプレート ページが開きます。リソースの詳細の一部は ARM テンプレートから入力されます。 空白のフィールドに次の情報を入力します。

    - **リソース グループ:** *contoso-rg* で始まります
    - **SQL 管理者ログイン:** labadmin
    - **SQL 管理者ログインのパスワード:** &lt;強力なパスワードを入力します&gt;

1. **[確認と作成]** を選択し、次に **[作成]** を選択します。 デプロイには 5 分程度かかります。

    ![画像 2](../images/dp-300-module-11-lab-02.png)

1. デプロイが完了したら、**[リソース グループに移動]** を選択します。 Azure リソース グループに移動します。これには、デプロイによって作成されたランダムな名前の **SQL Server** リソースが含まれています。

    ![図 3](../images/dp-300-module-11-lab-03.png)

ここでは、Azure Resource Manager テンプレート リンクを 1 回クリックするだけで、Azure SQL Server とデータベースの両方を簡単に作成できる方法を確認しました。
