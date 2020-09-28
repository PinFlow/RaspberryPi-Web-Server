# LEM＋Tomcat on RaspberryPi
ラズベリーパイにLEM(Linux, Nginx, MariaDB)＋Tomcatをセットアップして、adress 127.0.0.1にDBにある特定のデータを表示させてみた個人プロジェクトです。

## 1. 目的
サーバーをより理解するため、個人的にラズベリーパイでLinuxを使ってみました。

---
## 2.目標
- ラズベリーパイにLEM+Tomcatの環境構築
- http://localhost にTomcatページを表示
- http://localhost にデータベースのデータをJSPを使って表示

---
## 3. 環境構築
### 3.1 ラズベリーパイOSインストール
　<https://www.raspberrypi.org/downloads/>
#### 3.1.1 ラズベリーパイアップデート＆アップグレード
    sudo apt-get upgrade
    sudo apt-get update
### 3.2 開発道具Vimインストール
    sudo apt-get install vim
### 3.3 Nginxインストール
    sudo apt-get install nginx
### 3.4 MariaDBインストール
    sudo apt-get install mariadb-server
### 3.5 Tomcatインストール
    sudo pat-get install tomcat8
---
## 4. 環境セットアップ
### 4.1 puttyを使うためのssh活性化
ノートパソコンでラズベリーパイを操作したくてWi－Fiでラズベリーパイをネットワーク接続状態にしました。   
ラズベリーパイのオプションでsshを活性化しないとputtyは使えないので下のコードでsshを活性化します。   
    
    sudo raspi-config
    # option 5 - interfaciong options - p2 ssh
    # disableをenableに変更
    
ノートパソコンにputtyをインストールして、ipを入力したら遠隔でラズベリーパイを操作できます。   
下のコードをラズベリーパイに入力してipを確認しました。      
    
    ifconfig
    # ラズベリーパイでipを確認してノートパソコンのputtyにipを入力、
    # wlan0にあるinetにipが書かれています。
### 4.2 NginxにTomcatを接続
下の経路のNginx設定ファイルを修正

    vim /etc/nginx/site-available/default
    
Tomcat位置を指定して、serverブロック内のlocationにコードを追加する
    
    # Tomcat位置を指定
    upstream tomcat {
	    ip_hash;
	    server 127.0.0.1:8080;
    }
    
    # 80ポートに対する設定
    server {
        listen 80;
        listen [::]:80;
        root /var/www/html;

        # Add index.php to the list if you are using PHP
        index index.html index.htm index.nginx-debian.html;

        server_name _;
        charset utf-8;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                # try_files $uri $uri/ =404; //cssがロードできないときもあるのでコメントにする
                # proxy設定の追加
                proxy_pass http://tomcat; //tomcatサーバー
                proxy_set_header X-Real-IP $remote_addr; 
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Host $http_host;
   		}
    }
    #...中略
    
    # 443ポートに対する設定
    server {
        listen 443;
        listen [::]:443;
        ssl on;
        server_name _;

        ssl_certificate /etc/letsencrypt/live/8e7dcf95.ngrok.io/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/8e7dcf95.ngrok.io/privkey.pem;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                # try_files $uri $uri/ =404;
                # proxy設定の追加
                proxy_pass http://tomcat;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Host $http_host;
        }
    }
    
    # セーブした後、下のコードでnginxをリスタート
    service nginx restart
### 4.3 jarをダウンロードしてTomcatのフォルダに設置
<https://downloads.mariadb.com/Connectors/java/connector-java-2.7.0/>   
上記のURLでmariadb-java-client-2.7.0.jarをダウンロードして、/var/lib/tomcat8/lib経路にコピーします。
### 4.4 Java環境変数の設定
#### 4.4.1 javac経路を探す

    which javac
    # 結果：/usr/bin/javac
    # この経路はシンボリックリンクなので、実際の経路をreadlinkで探す
    
    readlink -f /usr/bin/javac
    # 結果：/usr/lib/jvm/java-11-openjdk-armhf/bin/javac
    # この経路を /etc/profileに環境変数で登録する
    
#### 4.4.2 シェル初期化ファイルの修正

    export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-armhf
    export PATH=$PATH:$JAVA_HOME/bin
    export CLASSPATH=.:$JAVA_HOME/lib/tools.jar:/var/lib/tomcat8/lib/mariadb-java-client-2.7.0.jar
    
ファイルをセーブしてシェル初期化ファイルを再びロードする

    source /etc/profile
    
### 4.5 MariaDB設定
#### 4.5.1 DB接続
    mysql -u root -p
#### 4.5.2 アカウント検索
    use mysql;
    select host, user, password from user;
#### 4.5.3 アカウント作成
    create user アカウント名@localhost identified by 'パスワード';
    create user アカウント名@'%' identified by 'パスワード';
#### 4.5.4 アカウント権限設定
    grant select on dbスキマ.* to 'アカウント名'@'localhost' identified by 'パスワード';
    grant select on dbスキマ.* to 'アカウント名'@'%' identified by 'パスワード';
    
#### 4.5.6 修正事項の繁栄
    mysql > flush privileges;
    
### 4.6 外部接続の設定
#### 4.6.1 コメント処理
    sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
を入力して50-server.cnfファイルでbind-adress=127.0.0.1部分をコメントに変更する
#### 4.6.2 ファイアウォールの設定
    sudo iptables -A INPUT -p tcp --dport 3306 -j ACCEPT
    sudo iptables -A OUTPUT -p tcp --dport 3306 -j ACCEPT
    sudo iptables-save
#### rootアカウントの権原設定
    sudo mysql -u root
    use mysql;
    grant all privileges on *.* to 'root'@'%' identified by 'パスワード';
    flush privileges;
#### mysqlリスタート
    sudo service mysql restart
---
## 5. JSPにデータベースのテーブルデータを表示
### 5.1 DBテーブルの作成
    # MariaDBに接続
    sudo mysql -u root
    
    # データベースの作成
    CREATE DATABASE test;
    
    # 使用するデータベースを選択
    use test;
    
    # データベースにテーブルを作成
    CREATE TABLE index_table(name varchar(30), dept varchar(30), Email varchar(30));
    
    # テーブルに値を入力
    INSERT INTO index_table VALUES('Jeong', 'Dev', 'abc@Gmail.com');
    
### 5.2 JSPの作成
    # 下の経路に移動するコード
    cd /var/lib/tomcat8/webapps/ROOT
    
    # vimでtest.jspを開く
    sudo vim test.jsp
    
    #vimのコマンド「a」: Append after cursor(カーソルの後ろに追加。作成モード)
    a
    
test.jspの内容
```
<%@ page language="java" 
         contentType="text/html; charset=UTF-8" 
         pageEncoding="UTF-8" %>
<%@ page import="java.sql.DriverManager" %>
<%@ page import="java.sql.Connection" %>
<%@ page import="java.sql.Statement" %>
<%@ page import="java.sql.ResultSet" %>
<%@ page import="java.sql.SQLException" %>
<% request.setCharacterEncoding("UTF-8"); %>

<html>
    <head>
        <title> Database Test </title>
    </head>

    <body>

        <% 
        Class.forName("org.mariadb.jdbc.Driver");
        Connection conn = null;
        Statement stmt = null;
        ResultSet rs = null;

        try{
        # jdbc driver, dbアカウント入力
        String jdbcDriver ="jdbc:mariadb://127.0.0.1:3306/test";
        String dbUser="アカウント名";
        String dbPass="パスワード";
        String query="select * from index_table";

        # データベースコネクション
        conn = DriverManager.getConnection(jdbcDriver,dbUser,dbPass);
        # Statement
            stmt = conn.createStatement();
            # Query実行
            rs = stmt.executeQuery(query);

            if(rs.next()) {
            %>
            <table>
                <tr>
                    # テーブルのカラムをrs.getString関数のパラメータタイプで参考
                    <td>name: <%=rs.getString("name")%></td>
                    <td>dept: <%=rs.getString("dept")%></td>
                    <td>Email: <%=rs.getString("Email")%></td>
                </tr>
            </table>
            <%
            } 

        }
        catch(SQLException ex) {
        %>
        Error Occurred :<%= ex.getMessage() %>
        <%
        }
        finally {
            if(rs!=null) try{rs.close();}catch(SQLException ex) {}
            if(stmt!=null) try{stmt.close();}catch(SQLException ex) {}
            if(conn!=null) try{conn.close();}catch(SQLException ex) {}
        }
        %>
    </body>
</html> 
```
### 5.3 作動の確認
    # test.jsp内で下のコードを入力してセーブしてからvim終了
    :wq
chromiumを開いて、adressにlocalhost:8080/test.jspを入力してindex_tableのデータが表示されるのか確認

### 5.4 ファイル名の変更
    # index.html -> index_old.htmlに変更
    sudo mv index.html index_old.html
    
    # 最初に表示させるページの名前はindex.htmlもしくはindex.jsp
    # test.jsp -> index.jspに変更してサーバページ(127.0.0.1、localhost、localhost:8080)に最初に表示させるようにできる
    sudo mv test.jsp index.jsp
---
   
