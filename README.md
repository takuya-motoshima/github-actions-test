# GitHub Actions 自動デプロイ

- [GitHub Actions 自動デプロイ](#github-actions-自動デプロイ)
    - [手順](#手順)
    - [エラー](#エラー)

## 手順
1. デプロイサーバでSSHキーペアを作成。
    ```sh
    ssh-keygen -t rsa -b 4096 -f id_rsa_github-actions-test
    ```
1. 公開鍵をGitHubリポジトリの Deploy keys として登録。
    Settings メニューから Deploy keys の登録画面を開く。  
    「Add deploy key」 のボタンをクリックし、「Title」に任意の名前、「Key」に先ほど作成したSSHキーペアの公開鍵を入力し、「Add Key」 ボタンを押下して、Deploy keys を登録。

    ![1.jpg](screencaps/1.jpg)

    「Allow write access」のチェックは、デプロイサーバーから GitHub リポジトリに対して push する必要がある場合にチェックを入れる。  
    今回は、GitHub → デプロイサーバー への一方通行なので、チェックは外しておく。
1. 必要な情報を Secrets に登録。  
    Secrets and variables → Actions メニューから Secrets の登録画面を開く。

    「New repository secret」ボタンを押下して、以下の内容を登録。

    <table width="100%">
        <thead>
            <tr>
                <th>項目名</th>
                <th>入力内容</th>
                <th>例</th>
            </tr>
        </thead>
        <tbody>
            <tr>
                <td>USERNAME</td>
                <td>デプロイサーバーのSSHユーザ名</td>
                <td>ec2-user</td>
            </tr>
            <tr>
                <td>HOST</td>
                <td>デプロイサーバのホスト名</td>
                <td>203.0.113.1</td>
            </tr>
            <tr>
                <td>PORT</td>
                <td>デプロイサーバのSSHポート番号</td>
                <td>22</td>
            </tr>
            <tr>
                <td>SSH_PRIVATE_KEY</td>
                <td>デプロイサーバーにSSH接続するための秘密鍵</td>
                <td></td>
            </tr>
            <tr>
                <td>DEPLOY_DIR</td>
                <td>デプロイ先のディレクトリ</td>
                <td>/var/www/html/test</td>
            </tr>
            </tr>
        </tbody>
    </table>

    ![2.jpg](screencaps/2.jpg)

    ※ここで指定した値は、あとで作成する workflow ファイルで、${{ secrets.[登録したSecretsの名前] }} と記載すると呼び出せる。
2. Workflow を作成。  
    リポジトリ内の Actions メニューに移動し、「Simple workflow」の「Configure」のボタンを押下してWorkflow のベースを取得。

    ![3.jpg](screencaps/3.jpg)


    GitHub上のエディタに、Workflow が記載された yml ファイルが展開される。
    ファイル名を変更できるので、今回は deploy.yml としておく。

    ![4.jpg](screencaps/4.jpg)

    今回は、main ブランチに push された時にデプロイするよう、deploy.yml を以下内容で置き換える。  
    置き換えたら、「Start Commit」ボタンを押下して、ファイルをリポジトリに保存。

    ```yml
    name: CI

    on:
      push:
        branches:
          - main

    jobs:
      build:
        runs-on: ubuntu-latest

        steps:
          - name: Deploy
            uses: appleboy/ssh-action@master
            with:
              host: ${{ secrets.HOST }}
              username: ${{ secrets.USERNAME }}
              port: ${{ secrets.PORT }}
              key: ${{ secrets.SSH_PRIVATE_KEY }}
              passphrase: ${{ secrets.SSH_PASS }}
              script: |
                cd ${{ secrets.DEPLOY_DIR }}
                git pull origin main
    ```

    Workflow の解説：
    - name: CI  
        Workflow の名前。デフォルトのまま使っているだけなので、任意のものに書き換えていない。
    - on:  
        アクションが実行されるタイミング。  
        今回は、main ブランチに push された時としている。
    - jobs:  
        実行する処理。
    - rans-on:
        処理の実行環境。
    - uses:  
        今回利用しているアクション。
    - with:  
        アクションに必要な値を設定し、script: 以下の処理を実行している。

1. 設定した Workflow の動作とエラーを確認する方法。  
    Workflow が正しく動作したかどうかは、GitHub リポジトリの Actions メニューから確認できる。

    ![5.jpg](screencaps/5.jpg)

2. テスト
    mainブランチに新規で「hello.txt」というファイルを作成し、デプロイサーバに反映されるか確認。

    ![6.jpg](screencaps/6.jpg)


    「hello.txt」が、自動でサーバにデプロイされている事を確認できた。
    ```sh
    $ ll
    total 8
    -rw-rw-r-- 1 ec2-user ec2-user 14 Jun 27 17:16 hello.txt
    -rw-rw-r-- 1 ec2-user ec2-user 59 Jun 27 17:10 README.md
    ``

## エラー
1. Buildでエラーが発生。
    ```sh
    err: fatal: not a git repository (or any of the parent directories): .git
    2023/06/27 08:05:10 Process exited with status 128
    ```

    デプロイサーバの該当ディレクトリにリポジトリをcloneしたら解決した。