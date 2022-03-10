[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Soliton-Analytics-Team/FullTextSearch_Groonga/blob/main/FullTextSearch_Groonga.ipynb)

# Colabで全文検索（その3：Groonga編）

各種全文検索ツールをColabで動かしてみるシリーズです。全7回の予定です。今回はGroongaです。

処理時間の計測はストレージのキャッシュとの兼ね合いがあるので、2回測ります。2回目は全てがメモリに載った状態での性能評価になります。ただ1回目もデータを投入した直後なので、メモリに載ってしまっている可能性があります。

## 準備

まずは検索対象のテキストを日本語wikiから取得して、Google Driveに保存します。（※ Google Driveに約１GBの空き容量が必要です。以前のデータが残っている場合は取得せず再利用します。）

Google Driveのマウント


```python
from google.colab import drive
drive.mount('/content/drive')
```

    Mounted at /content/drive


jawikiの取得とjson形式に変換。90分ほど時間がかかります。他の全文検索シリーズでも同じデータを使うので、他の記事も試す場合は wiki.json.bz2 を捨てずに残しておくことをおすすめします。


```shell
%%time
%cd /content/
import os
if not os.path.exists('/content/drive/MyDrive/wiki.json.bz2'):
    !wget https://dumps.wikimedia.org/jawiki/latest/jawiki-latest-pages-articles.xml.bz2
    !pip install wikiextractor
    !python -m wikiextractor.WikiExtractor --no-templates --processes 4 --json -b 10G -o - jawiki-latest-pages-articles.xml.bz2 | bzip2 -c > /content/drive/MyDrive/wiki.json.bz2
```

json形式に変換されたデータを確認


```python
import json
import bz2

with bz2.open('/content/drive/MyDrive/wiki.json.bz2', 'rt', encoding='utf-8') as fin:
    for n, line in enumerate(fin):
        data = json.loads(line)
        print(data['title'].strip(), data['text'].replace('\n', '')[:40], sep='\t')
        if n == 5:
            break
```

    アンパサンド	アンパサンド（&amp;, ）は、並立助詞「…と…」を意味する記号である。ラテン
    言語	言語（げんご）は、広辞苑や大辞泉には次のように解説されている。『日本大百科事典』
    日本語	 日本語（にほんご、にっぽんご）は、日本国内や、かつての日本領だった国、そして日
    地理学	地理学（ちりがく、、、伊：geografia、）は、。地域や空間、場所、自然環境
    EU (曖昧さ回避)	EU
    国の一覧	国の一覧（くにのいちらん）は、世界の独立国の一覧。対象.国際法上国家と言えるか否


## Groongaのインストール


```shell
!sudo apt-get -y install software-properties-common
!sudo add-apt-repository -y universe
!sudo add-apt-repository -y ppa:groonga/ppa
!sudo apt-get update
!sudo apt-get jq
!sudo apt-get -y install groonga
!groonga --version
```

    Reading package lists... Done
    Building dependency tree       
    Reading state information... Done
    software-properties-common is already the newest version (0.96.24.32.18).
    The following package was automatically installed and is no longer required:
      libnvidia-common-470
    Use 'sudo apt autoremove' to remove it.
    0 upgraded, 0 newly installed, 0 to remove and 39 not upgraded.
    'universe' distribution component is already enabled for all sources.
    Get:1 https://cloud.r-project.org/bin/linux/ubuntu bionic-cran40/ InRelease [3,626 B]
    Ign:2 https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64  InRelease
    Get:3 http://security.ubuntu.com/ubuntu bionic-security InRelease [88.7 kB]
    Get:4 http://ppa.launchpad.net/c2d4u.team/c2d4u4.0+/ubuntu bionic InRelease [15.9 kB]
    Hit:5 http://archive.ubuntu.com/ubuntu bionic InRelease
    Ign:6 https://developer.download.nvidia.com/compute/machine-learning/repos/ubuntu1804/x86_64  InRelease
    Get:7 https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64  Release [696 B]
    Hit:8 https://developer.download.nvidia.com/compute/machine-learning/repos/ubuntu1804/x86_64  Release
    Get:9 https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64  Release.gpg [836 B]
    Get:10 http://archive.ubuntu.com/ubuntu bionic-updates InRelease [88.7 kB]
    Get:11 https://cloud.r-project.org/bin/linux/ubuntu bionic-cran40/ Packages [76.8 kB]
    Hit:12 http://ppa.launchpad.net/cran/libgit2/ubuntu bionic InRelease
    Get:14 http://archive.ubuntu.com/ubuntu bionic-backports InRelease [74.6 kB]
    Get:15 https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64  Packages [931 kB]
    Hit:16 http://ppa.launchpad.net/deadsnakes/ppa/ubuntu bionic InRelease
    Get:17 http://ppa.launchpad.net/graphics-drivers/ppa/ubuntu bionic InRelease [21.3 kB]
    Get:18 http://security.ubuntu.com/ubuntu bionic-security/main amd64 Packages [2,596 kB]
    Get:19 http://ppa.launchpad.net/groonga/ppa/ubuntu bionic InRelease [21.3 kB]
    Get:20 http://archive.ubuntu.com/ubuntu bionic-updates/restricted amd64 Packages [840 kB]
    Get:21 http://ppa.launchpad.net/c2d4u.team/c2d4u4.0+/ubuntu bionic/main Sources [1,827 kB]
    Get:22 http://archive.ubuntu.com/ubuntu bionic-updates/main amd64 Packages [3,036 kB]
    Get:23 http://security.ubuntu.com/ubuntu bionic-security/restricted amd64 Packages [806 kB]
    Get:24 http://security.ubuntu.com/ubuntu bionic-security/universe amd64 Packages [1,474 kB]
    Get:25 http://archive.ubuntu.com/ubuntu bionic-updates/universe amd64 Packages [2,252 kB]
    Get:26 http://ppa.launchpad.net/c2d4u.team/c2d4u4.0+/ubuntu bionic/main amd64 Packages [936 kB]
    Get:27 http://ppa.launchpad.net/graphics-drivers/ppa/ubuntu bionic/main amd64 Packages [42.8 kB]
    Get:28 http://ppa.launchpad.net/groonga/ppa/ubuntu bionic/main amd64 Packages [5,774 B]
    Fetched 15.1 MB in 4s (4,271 kB/s)
    Reading package lists... Done
    Hit:1 http://security.ubuntu.com/ubuntu bionic-security InRelease
    Hit:2 https://cloud.r-project.org/bin/linux/ubuntu bionic-cran40/ InRelease
    Ign:3 https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64  InRelease
    Hit:4 http://ppa.launchpad.net/c2d4u.team/c2d4u4.0+/ubuntu bionic InRelease
    Hit:5 http://archive.ubuntu.com/ubuntu bionic InRelease
    Ign:6 https://developer.download.nvidia.com/compute/machine-learning/repos/ubuntu1804/x86_64  InRelease
    Hit:7 https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64  Release
    Hit:8 http://archive.ubuntu.com/ubuntu bionic-updates InRelease
    Hit:9 https://developer.download.nvidia.com/compute/machine-learning/repos/ubuntu1804/x86_64  Release
    Hit:10 http://archive.ubuntu.com/ubuntu bionic-backports InRelease
    Hit:11 http://ppa.launchpad.net/cran/libgit2/ubuntu bionic InRelease
    Hit:12 http://ppa.launchpad.net/deadsnakes/ppa/ubuntu bionic InRelease
    Hit:14 http://ppa.launchpad.net/graphics-drivers/ppa/ubuntu bionic InRelease
    Hit:16 http://ppa.launchpad.net/groonga/ppa/ubuntu bionic InRelease
    Reading package lists... Done
    E: Invalid operation jq
    Reading package lists... Done
    Building dependency tree       
    Reading state information... Done
    The following package was automatically installed and is no longer required:
      libnvidia-common-470
    Use 'sudo apt autoremove' to remove it.
    The following additional packages will be installed:
      groonga-bin groonga-doc groonga-plugin-suggest javascript-common libgroonga0
      libjs-jquery libjs-underscore libmsgpackc2
    Suggested packages:
      apache2 | lighttpd | httpd
    The following NEW packages will be installed:
      groonga groonga-bin groonga-doc groonga-plugin-suggest javascript-common
      libgroonga0 libjs-jquery libjs-underscore libmsgpackc2
    0 upgraded, 9 newly installed, 0 to remove and 78 not upgraded.
    Need to get 6,708 kB of archives.
    After this operation, 69.0 MB of additional disk space will be used.
    Get:1 http://archive.ubuntu.com/ubuntu bionic/universe amd64 libmsgpackc2 amd64 2.1.5-1 [14.3 kB]
    Get:2 http://ppa.launchpad.net/groonga/ppa/ubuntu bionic/main amd64 libgroonga0 amd64 12.0.1-1.ubuntu18.04.1 [1,822 kB]
    Get:3 http://archive.ubuntu.com/ubuntu bionic-updates/main amd64 libjs-underscore all 1.8.3~dfsg-1ubuntu0.1 [60.4 kB]
    Get:4 http://archive.ubuntu.com/ubuntu bionic/main amd64 libjs-jquery all 3.2.1-1 [152 kB]
    Get:5 http://archive.ubuntu.com/ubuntu bionic/main amd64 javascript-common all 11 [6,066 B]
    Get:6 http://ppa.launchpad.net/groonga/ppa/ubuntu bionic/main amd64 groonga-bin amd64 12.0.1-1.ubuntu18.04.1 [424 kB]
    Get:7 http://ppa.launchpad.net/groonga/ppa/ubuntu bionic/main amd64 groonga-plugin-suggest amd64 12.0.1-1.ubuntu18.04.1 [70.6 kB]
    Get:8 http://ppa.launchpad.net/groonga/ppa/ubuntu bionic/main amd64 groonga-doc all 12.0.1-1.ubuntu18.04.1 [4,151 kB]
    Get:9 http://ppa.launchpad.net/groonga/ppa/ubuntu bionic/main amd64 groonga amd64 12.0.1-1.ubuntu18.04.1 [8,564 B]
    Fetched 6,708 kB in 3s (2,094 kB/s)
    debconf: unable to initialize frontend: Dialog
    debconf: (No usable dialog-like program is installed, so the dialog based frontend cannot be used. at /usr/share/perl5/Debconf/FrontEnd/Dialog.pm line 76, <> line 9.)
    debconf: falling back to frontend: Readline
    debconf: unable to initialize frontend: Readline
    debconf: (This frontend requires a controlling tty.)
    debconf: falling back to frontend: Teletype
    dpkg-preconfigure: unable to re-open stdin: 
    Selecting previously unselected package libmsgpackc2:amd64.
    (Reading database ... 155320 files and directories currently installed.)
    Preparing to unpack .../0-libmsgpackc2_2.1.5-1_amd64.deb ...
    Unpacking libmsgpackc2:amd64 (2.1.5-1) ...
    Selecting previously unselected package libgroonga0:amd64.
    Preparing to unpack .../1-libgroonga0_12.0.1-1.ubuntu18.04.1_amd64.deb ...
    Unpacking libgroonga0:amd64 (12.0.1-1.ubuntu18.04.1) ...
    Selecting previously unselected package groonga-bin.
    Preparing to unpack .../2-groonga-bin_12.0.1-1.ubuntu18.04.1_amd64.deb ...
    Unpacking groonga-bin (12.0.1-1.ubuntu18.04.1) ...
    Selecting previously unselected package groonga-plugin-suggest.
    Preparing to unpack .../3-groonga-plugin-suggest_12.0.1-1.ubuntu18.04.1_amd64.deb ...
    Unpacking groonga-plugin-suggest (12.0.1-1.ubuntu18.04.1) ...
    Selecting previously unselected package libjs-underscore.
    Preparing to unpack .../4-libjs-underscore_1.8.3~dfsg-1ubuntu0.1_all.deb ...
    Unpacking libjs-underscore (1.8.3~dfsg-1ubuntu0.1) ...
    Selecting previously unselected package libjs-jquery.
    Preparing to unpack .../5-libjs-jquery_3.2.1-1_all.deb ...
    Unpacking libjs-jquery (3.2.1-1) ...
    Selecting previously unselected package groonga-doc.
    Preparing to unpack .../6-groonga-doc_12.0.1-1.ubuntu18.04.1_all.deb ...
    Unpacking groonga-doc (12.0.1-1.ubuntu18.04.1) ...
    Selecting previously unselected package groonga.
    Preparing to unpack .../7-groonga_12.0.1-1.ubuntu18.04.1_amd64.deb ...
    Unpacking groonga (12.0.1-1.ubuntu18.04.1) ...
    Selecting previously unselected package javascript-common.
    Preparing to unpack .../8-javascript-common_11_all.deb ...
    Unpacking javascript-common (11) ...
    Setting up libjs-jquery (3.2.1-1) ...
    Setting up libjs-underscore (1.8.3~dfsg-1ubuntu0.1) ...
    Setting up groonga-doc (12.0.1-1.ubuntu18.04.1) ...
    Setting up javascript-common (11) ...
    Setting up libmsgpackc2:amd64 (2.1.5-1) ...
    Setting up libgroonga0:amd64 (12.0.1-1.ubuntu18.04.1) ...
    Setting up groonga-bin (12.0.1-1.ubuntu18.04.1) ...
    Setting up groonga-plugin-suggest (12.0.1-1.ubuntu18.04.1) ...
    Setting up groonga (12.0.1-1.ubuntu18.04.1) ...
    Processing triggers for libc-bin (2.27-3ubuntu1.3) ...
    /sbin/ldconfig.real: /usr/local/lib/python3.7/dist-packages/ideep4py/lib/libmkldnn.so.0 is not a symbolic link
    
    Groonga 12.0.1 [linux-gnu,x86_64,utf8,match-escalation-threshold=0,nfkc,mecab,message-pack,mruby,onigmo,zlib,lz4,zstandard,epoll,rapidjson]
    
    configure options: < '--build=x86_64-linux-gnu' '--prefix=/usr' '--includedir=${prefix}/include' '--mandir=${prefix}/share/man' '--infodir=${prefix}/share/info' '--sysconfdir=/etc' '--localstatedir=/var' '--disable-silent-rules' '--libdir=${prefix}/lib/x86_64-linux-gnu' '--libexecdir=${prefix}/lib/x86_64-linux-gnu' '--disable-maintainer-mode' '--disable-dependency-tracking' '--with-munin-plugins' '--enable-mruby' 'build_alias=x86_64-linux-gnu' 'CFLAGS=-g -O2 -fdebug-prefix-map=/build/groonga-Djqvje/groonga-12.0.1=. -fstack-protector-strong -Wformat -Werror=format-security' 'LDFLAGS=-Wl,-Bsymbolic-functions -Wl,-z,relro -Wl,-z,now' 'CPPFLAGS=-Wdate-time -D_FORTIFY_SOURCE=2' 'CXXFLAGS=-g -O2 -fdebug-prefix-map=/build/groonga-Djqvje/groonga-12.0.1=. -fstack-protector-strong -Wformat -Werror=format-security'>


## DB作成

場所を指定してDBを作成します。


```shell
!echo "" | groonga -n /tmp/db
```

作成されたことを確認します。


```shell
!groonga /tmp/db status
```

    [[0,1646667764.569797,6.771087646484375e-05],{"alloc_count":12132,"starttime":1646667764,"start_time":1646667764,"uptime":0,"version":"12.0.1","n_queries":0,"cache_hit_rate":0.0,"command_version":1,"default_command_version":1,"max_command_version":3,"n_jobs":0,"features":{"nfkc":true,"mecab":true,"message_pack":true,"mruby":true,"onigmo":true,"zlib":true,"lz4":true,"zstandard":true,"kqueue":false,"epoll":true,"poll":false,"rapidjson":true,"apache_arrow":false,"xxhash":false}}]


DBにテーブルとカラムを作成します。


```shell
!groonga /tmp/db table_create --name wiki_jp --flags TABLE_HASH_KEY --key_type ShortText
!groonga /tmp/db column_create --table wiki_jp --name body --type Text
```

    [[0,1646667764.700085,0.006307125091552734],true]
    [[0,1646667764.820107,0.01014184951782227],true]


作成されたテーブルを確認します。


```shell
!groonga /tmp/db select --table wiki_jp
```

    [[0,1646667764.935338,0.0009636878967285156],[[[0],[["_id","UInt32"],["_key","ShortText"],["body","Text"]]]]]


## データのインポート

データを解凍して50万件分を書き出します。


```python
import json
import bz2
from tqdm.notebook import tqdm

limit = 500000
with open('/content/load.txt', 'w') as fout:
    print('load --table wiki_jp\n[', file=fout)
    with bz2.open('/content/drive/MyDrive/wiki.json.bz2', 'rt', encoding='utf-8') as fin:
        n = 0
        for line in tqdm(fin, total=limit*1.5):
            data = json.loads(line)
            title = data['title'].strip()
            body = data['text'].replace('\n', '')
            if len(title) > 0 and len(body) > 0:
                print(json.dumps({"_key":title, "body":body}, ensure_ascii=False), ',', file=fout)
                n += 1
            if n == limit:
                break
    print(']', file=fout)
```


      0%|          | 0/750000.0 [00:00<?, ?it/s]


書き出したデータを読み込ませます。


```shell
%%time
!groonga /tmp/db < /content/load.txt
```

    [[0,1646668165.571066,23.59745812416077],500000]
    CPU times: user 195 ms, sys: 33.2 ms, total: 228 ms
    Wall time: 23.8 s


登録件数を確認します。


```shell
!groonga /tmp/db select wiki_jp --limit 0 
```

    [[0,1646668189.474985,0.001855611801147461],[[[500000],[["_id","UInt32"],["_key","ShortText"],["body","Text"]]]]]


## インデックスを使わない検索

bodyに「日本語」を含むレコードの数を取得します。


```shell
%%time
!groonga /tmp/db select wiki_jp --query body:@日本語 --limit 0
```

    [[0,1646379721.581422,27.47038435935974],[[[17006],[["_id","UInt32"],["_key","ShortText"],["body","Text"]]]]]
    CPU times: user 215 ms, sys: 28.8 ms, total: 244 ms
    Wall time: 27.7 s


bodyに「日本語」を含むレコードを取得します。


```shell
%%time
!groonga /tmp/db select wiki_jp --query body:@日本語 --limit 100000 > /dev/null
```

    CPU times: user 238 ms, sys: 30.3 ms, total: 268 ms
    Wall time: 29.8 s


2回目


```shell
%%time
!groonga /tmp/db select wiki_jp --query body:@日本語 --limit 100000 > /dev/null
```

    CPU times: user 242 ms, sys: 27.5 ms, total: 270 ms
    Wall time: 29.1 s


インデックスを使わない検索では、ヒットする件数を算出するにもbodyを読み込んでその中に「日本語」が含まれるか確認する必要があるので、レコードの数を取得するのとレコードを取得するので、処理時間の差があまりありません。

インデックスを使わない場合、MySQLで14秒、PostgreSQLで15秒ですから、Groongaは相当遅いです。

## インデックスを使った検索

インデックスを作成します。


```shell
!groonga /tmp/db table_create --name Terms --flags TABLE_PAT_KEY --key_type ShortText --default_tokenizer TokenBigram --normalizer NormalizerAuto
```

    [[0,1646668189.671915,0.00914764404296875],true]



```shell
%%time
!groonga /tmp/db column_create --table Terms --name wiki_body --flags "COLUMN_INDEX|WITH_POSITION" --type wiki_jp --source body
```

    [[0,1646668189.800892,413.5416598320007],true]
    CPU times: user 3.18 s, sys: 432 ms, total: 3.61 s
    Wall time: 6min 53s


bodyに「日本語」を含むレコードの数を取得します。


```shell
%%time
!groonga /tmp/db select wiki_jp --query body:@日本語 --limit 0
```

    [[0,1646378101.153079,0.02551054954528809],[[[17006],[["_id","UInt32"],["_key","ShortText"],["body","Text"]]]]]
    CPU times: user 11.1 ms, sys: 9.08 ms, total: 20.2 ms
    Wall time: 324 ms


レコードのbodyにアクセスする必要がなく、インデックスのみを参照しているため、非常に速いです。

bodyに「日本語」を含むレコードを取得します。


```shell
%%time
!groonga /tmp/db select wiki_jp --query body:@日本語 --limit 100000 > /dev/null
```

    CPU times: user 19.8 ms, sys: 10.2 ms, total: 30 ms
    Wall time: 1.72 s


2回目


```shell
%%time
!groonga /tmp/db select wiki_jp --query body:@日本語 --limit 1000000 > /dev/null
```

    CPU times: user 25.9 ms, sys: 5.27 ms, total: 31.2 ms
    Wall time: 1.73 s


データがメモリに乗った状態で、MySQLが8秒、PostgreSQLが9秒ですから、Groongaは相当速いです。全文検索特化の面目躍如です。

参考までに、bodyに「日本語」を含むレコードのタイトルとスコアのみを取得します。


```shell
%%time
!groonga /tmp/db select wiki_jp --query body:@日本語 --limit 1000000 --output_columns _key,_score > /dev/null
```

    CPU times: user 7.02 ms, sys: 6.86 ms, total: 13.9 ms
    Wall time: 220 ms


9割近くが文章の読み込みの時間であることが分かります。

## おまけ

以下のようにすると「日本語」と「表現」を含むレコードが検索されます。


```shell
!groonga /tmp/db select wiki_jp --match_columns body --query '"日本語 表現"' --output_columns body --limit 0
!groonga /tmp/db select wiki_jp --match_columns body --query '"表現 日本語"' --output_columns body --limit 0
```

    [[0,1646668653.196783,0.02809000015258789],[[[3267],[["body","Text"]]]]]
    [[0,1646668653.399063,0.02009105682373047],[[[3267],[["body","Text"]]]]]


以下のようにすると、「日本語」と「表現」が並んで出現するレコードを検索します。


```shell
!groonga /tmp/db select wiki_jp --match_columns body --query '"\"日本語 表現\""' --output_columns body --limit 0
```

    [[0,1646668660.337281,0.01356887817382812],[[[37],[["body","Text"]]]]]


従って順序を変えると異なる結果になります。


```shell
!groonga /tmp/db select wiki_jp --match_columns body --query '"\"表現 日本語\""' --output_columns body --limit 0
```

    [[0,1646668665.709357,0.0197443962097168],[[[0],[["body","Text"]]]]]


以下は「日本語表記」を検索した結果です。今回は同数ですが、「日本語 表現」では「日本語」と「表現」の間に空白があってもなくてもマッチしますが、下記では空白を挟むとマッチしません。


```shell
!groonga /tmp/db select wiki_jp --match_columns body --query '日本語表現' --output_columns body --limit 0
```

    [[0,1646668680.297208,0.01164722442626953],[[[37],[["body","Text"]]]]]


以下では「日本語」もしくは「表現」を含むレコードが検索されます。


```shell
%%time
!groonga /tmp/db select wiki_jp --match_columns body --query '"日本語 OR 表現"' --output_columns body --limit 0
```

    [[0,1646668701.140637,0.02257537841796875],[[[34825],[["body","Text"]]]]]
    CPU times: user 10 ms, sys: 8.02 ms, total: 18.1 ms
    Wall time: 220 ms

