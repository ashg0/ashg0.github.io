---
layout: post
title:  "HTTP Serverをfork,thread,pollで実装"
date:   2022-02-20 12:00:00
---
[先週](https://ashg0.github.io/2022/02/12/standard-linux.html)の続き

15章以降のhttpサーバーはacceptする毎にforkする素朴な実装となっている。ApacheやNginxといった実際に商用レベルで利用されるサーバーはより効率的で並列処理可能な実装であるとのことだったので調べた。

結果として、知りたいことが正しく記述してある記事を見つけたので、これを真似して作成したhttpサーバーを改造してみる。
#### [Zerohttpd](https://unixism.net/2019/04/linux-applications-performance-introduction/)
やはりインターネット上に良質なリソースはいくらでもあるので、いかに発見して利用するかが学習効率につながるよなーと。知の高速道路というか整備されていない未舗装路をなんとか乗りこなしていきたい。

- [Code](https://github.com/ashg0/standard-linux/tree/master/concurrency)

#### **1. Iterative**  
そもそもforkしない実装なので、同時にさばけるコネクションは1つだけ。  
ふつうのLinuxプログラミングのhttpd2.cからforkを取り除いて作成。
```
static void
server_main(int server_fd, char *docroot)
{
    for (;;) {
        struct sockaddr_storage addr;
        socklen_t addrlen = sizeof addr;
        int sock;
        int pid;

        sock = accept(server_fd, (struct sockaddr*)&addr, &addrlen);
        if (sock < 0) log_exit("accept(2) failed: %s", strerror(errno));

        FILE *inf = fdopen(sock, "r");
        FILE *outf = fdopen(sock, "w");

        service(inf, outf, docroot);
        close(sock);
    }
}
```

#### **2. Forking**  
httpd2.cそのもの  
Iterativeと比較すると、accept後にforkが入る  
他にもSIGCHLDのハンドリングが必要になってくる。
```
static void
server_main(int server_fd, char *docroot)
{
    for (;;) {
        struct sockaddr_storage addr;
        socklen_t addrlen = sizeof addr;
        int sock;
        int pid;

        sock = accept(server_fd, (struct sockaddr*)&addr, &addrlen);
        if (sock < 0) log_exit("accept(2) failed: %s", strerror(errno));
		pid = fork();
		if (pid < 0) exit(3);
		if (pid == 0) {
        	FILE *inf = fdopen(sock, "r");
        	FILE *outf = fdopen(sock, "w");

        	service(inf, outf, docroot);
			exit(0);
		}
        close(sock);
    }
}
```

#### **3. Preforked**  
accept以前にさらにforkしておくため、最初に起動した親プロセス、acceptする子プロセス、accept後にforkしてリクエストをハンドリングする孫プロセス、となる。

親プロセスは以下create_child内で子プロセスをforkした後はpause()に入って何もしない（シグナルハンドリングはする）、子プロセスはloopでacceptする、孫プロセスは1リクエスト処理したらexit。
```
int
main(int argc, char *argv[])
{
//中略
    for(int i=0; i < PREFORK_CHILDREN; i++)
        pids[i] = create_child(i, server_fd, docroot);

    for(;;)
        pause();
}
```
```
static pid_t
create_child(int index, int listening_socket, char *docroot) {
    pid_t pid;

    pid = fork();
    if (pid < 0) exit(3);
    else if (pid > 0)
        return (pid);
    printf("Server %d(pid: %ld) starting\n", index, (long)getpid());
    server_main(listening_socket, docroot);
}
```
元記事のパフォーマンスを見ると、Preforkにすると単純なForkと比べてconcurrency 1000で5倍くらいのreq/secが出るらしい。
また[Apache MPM prefork](https://httpd.apache.org/docs/2.4/en/mod/prefork.html)がこの実装となっている。

- **The Thundering Herd**  
ざっくり理解したつもりで説明すると、多数のプロセス/スレッドがAcceptで同じイベントを待っている状態でリクエストが来ると、全てのプロセス/スレッドを一旦Wakeして1つ選んで他はBlockに戻るのがオーバーヘッドになるらしい。このPreforkの実装は特に何も気にしていないので、発生すると思いきや、Linuxでは[This](https://stackoverflow.com/questions/2213779/does-the-thundering-herd-problem-exist-on-linux-anymore)や[This](https://uwsgi-docs.readthedocs.io/en/latest/articles/SerializingAccept.html)の通り影響ないとのこと。

#### **4. Threaded**  
スレッドを使うが、流れはForkと同じとなる。  
acceptしたらpthread_createでスレッドを作成する。子スレッドには関数ポインタserviceを渡して実行し、親スレッドはloopしてacceptでBlockされてリクエストを待つ。
```
static void
server_main(int server_fd, char *docroot)
{
    for (;;) {
        struct sockaddr_storage addr;
        socklen_t addrlen = sizeof addr;
        int sock;
        pthread_t tid;
        struct service_args args;
        args.docroot = docroot;

        sock = accept(server_fd, (struct sockaddr*)&addr, &addrlen);
        if (sock < 0) log_exit("accept(2) failed: %s", strerror(errno));

        args.inf = fdopen(sock, "r");
        args.outf = fdopen(sock, "w");

        pthread_create(&tid, NULL, &service, (void *)&args);
    }
}
```
serviceをポインタに変更し、引数も構造体で渡す(のが正解なのかはわからないが、StackOverflowにあった)。
```
struct service_args {
    FILE *inf;
    FILE *outf;
    char *docroot;
};
```
```
static void
*service(void *arguments)
{
    struct HTTPRequest *req;
    struct service_args *args = (struct service_args *)arguments;
    FILE *in = args->inf;
    FILE *out = args->outf;
    char *docroot = args->docroot;

    pthread_detach(pthread_self());

    req = read_request(in);
    respond_to(req, out, docroot);
    free_request(req);

    fclose(in);
    fclose(out);
    return(NULL);
}
```
pthread_detach()はpthread_join()を呼ばなくてもteminate時にリソース開放可能にする。pthread_join()はwait()相当だと理解した。  
FILEストリームをクローズして、NULLを返して関数終わり=thread終わり となる。  
パフォーマンスに関してはForkには勝るがPreforkには劣る。Forkとの差はプロセス生成とスレッド生成のオーバーヘッドの差ということだと思う。

#### **5. Prethreaded**
mainでTHREADS_COUNT分creat_threadを呼ぶ。  
server_fdはGlobalになっている
```
int
main(int argc, char *argv[])
{
//略
    server_fd = listen_socket(port);
//略
    for (int i = 0; i < THREADS_COUNT; ++i) {
        create_thread(i, docroot);
    }

    for(;;)
        pause();
}
```
create_threadはpthread_createにserver_main渡して呼ぶだけ
```
static void
create_thread(int index, char *docroot) {

   pthread_create(&threads[index], NULL, &server_main, NULL);

}
```
server_mainではaccept->serviceをloopするだけ
```
static void
*server_main(void *targ)
{
    char *docroot = (char *) targ;
    for (;;) {
        struct sockaddr_storage addr;
        socklen_t addrlen = sizeof addr;
        int sock;

        pthread_mutex_lock(&mlock);
        sock = accept(server_fd, (struct sockaddr*)&addr, &addrlen);
        if (sock < 0) log_exit("accept(2) failed: %s", strerror(errno));
        pthread_mutex_unlock(&mlock);

        FILE *inf = fdopen(sock, "r");
        FILE *outf = fdopen(sock, "w");

        service(inf, outf, docroot);
    }
}
```
pthread_mutex_lockを呼ぶことで、1つのスレッドだけがmutex(と言われても実態は何なんだ…)を獲得し、以下のコードへ進む。つまり、acceptでblockされるのは1スレッドだけになり、Thundering Herdを回避することができる。unlockを呼ぶことで、他のスレッドがmutexを獲得可能になり、acceptを呼ぶことができるようになる。  
[Apache MPM worker](https://httpd.apache.org/docs/2.4/en/mod/worker.html)ではForkした複数の子プロセスが、それぞれ複数スレッドを作成する。

パフォーマンス(req/sec)としては、Forkと比較してメモリをshareするので軽量になり、スレッド作成はフォークよりもオーバーヘッドが少ない。  
単純なThreadと比較すると40%性能向上がみられるが、Preforkと比較すると同程度となる。これはLinuxにおいてプロセスとスレッドは同じようにスケジュールされるので、オーバーヘッドが大差ないということらしい。差分があるのはスレッド/プロセスの新規作成時となる。ConcurrencyにおいてはPrethredに分があり、元記事の条件だとPreforkは5000concurentまで性能を維持したのに対し、Prethredは11000concurentまで維持できた。

#### **6. Poll**
ここからが本題
