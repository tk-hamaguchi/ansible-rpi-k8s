RaspberryPiでおうちkubernetesをやるためのAnsiblePlayBook
========

事前準備
----

### ホスト端末にansibleとsshpassをインストールする

```sh
brew install ansible
brew install http://git.io/sshpass.rb
```


### Raspbian入りディスクイメージの作成と起動

* Raspbianの最新版をダウンロードする（記録時点では`2019-04-08`版が最新）
* MacにMicroSDを挿入する
* `diskutil`でマウント先を確認する（この環境では`disk4`）
    ```sh
    diskutil list
    ```
    ```plain
    /dev/disk4 (external, physical):
       #:                       TYPE NAME                    SIZE       IDENTIFIER
       0:     FDisk_partition_scheme                        *31.9 GB    disk4
       1:             Windows_FAT_32 NO NAME                 31.9 GB    disk4s1
    ```
* 最初`NO NAME`として自動マウントされるので、アンマウントする
    ```sh
    diskutil umount /Volumes/NO\ NAME
    ```
    ```plain
    Volume NO NAME on disk4s1 unmounted
    ```
* ダウンロードしたイメージをMicroSDに書き込む
    ```sh
    sudo dd bs=4m if=~/Downloads/2019-04-08-raspbian-stretch-lite.img of=/dev/disk4
    ```
    ```plain
    430+0 records in
    430+0 records out
    1803550720 bytes transferred in 631.009384 secs (2858200 bytes/sec)
    ```
* 起動時にsshでアクセスできるようにsshdを起動するように設定しておく
    ```sh
    touch /Volumes/boot/ssh
    ```
* `boot`が自動マウントされているのでアンマウントする
    ```sh
    diskutil umount /Volumes/boot
    ```
    ```plain
    Volume boot on disk4s1 unmounted
    ```
* IPアドレスがわからないときはHDMIモニターを繋ぐ
* RaspberryPiへSDカードを入れて起動する



セットアップの実行
--------

* hostsファイルの作成
    ```sh
    echo 'ansible_host: 192.168.0.11' > ouchi/host_vars/rpi-k8s-master-01.yml
    echo 'ansible_host: 192.168.0.21' > ouchi/host_vars/rpi-k8s-worker-01.yml
    echo 'ansible_host: 192.168.0.22' > ouchi/host_vars/rpi-k8s-worker-02.yml
    ```
* Playbookの実行
    ```sh
    ansible-playbook -i ouchi rpi-k8s.yml
    ```
* MasterNodeの有効化（今は手動）
    ```sh
    ssh rpi-k8s-master-01 -l pi
    sudo kubeadm init --pod-network-cidr=10.244.0.0/16
    ```
    ```plain
    Your Kubernetes control-plane has initialized successfully!
    
    To start using your cluster, you need to run the following as a regular user:
    
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      https://kubernetes.io/docs/concepts/cluster-administration/addons/
    
    Then you can join any number of worker nodes by running the following on each as root:
    
    kubeadm join 192.168.0.11:6443 --token 50205d7fd7923acf4e5814o \
        --discovery-token-ca-cert-hash sha256:7f52673218569a56fdbe499580beb7436d3cdc4452b746c9cc3e0ab2c2e93912
    ```
* 手順通りログインユーザーのホームディレクトリに設定ファイルをコピー
    ```sh
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
* CNIとしてflannelを入れてみる（インストール元は[公式ドキュメント](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network)参照）
    ```sh
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
    ```
* WorkerNodeをクラスタに参画させる（コマンドはMasterNodeの有効化の出力を利用する）
    ```sh
    ssh rpi-k8s-worker-01 -l pi
    sudo kubeadm join 192.168.0.11:6443 --token 50205d7fd7923acf4e5814o --discovery-token-ca-cert-hash sha256:7f52673218569a56fdbe499580beb7436d3cdc4452b746c9cc3e0ab2c2e93912
    exit
    ```
    ```sh
    ssh rpi-k8s-worker-02 -l pi
    sudo kubeadm join 192.168.0.11:6443 --token 50205d7fd7923acf4e5814o --discovery-token-ca-cert-hash sha256:7f52673218569a56fdbe499580beb7436d3cdc4452b746c9cc3e0ab2c2e93912
    exit
    ```
* MasterNodeでノードがReadyになるまで待機
    ```sh
    kubectl get nodes -w
    ```
    ```plain
    NAME                STATUS     ROLES    AGE     VERSION
    rpi-k8s-master-01   Ready      master   7m38s   v1.14.1
    rpi-k8s-worker-01   NotReady   <none>   28s     v1.14.1
    rpi-k8s-worker-02   NotReady   <none>   10s     v1.14.1
    rpi-k8s-worker-01   Ready      <none>   31s     v1.14.1
    rpi-k8s-worker-02   Ready      <none>   31s     v1.14.1
    Ctrl+c
    ```
* MasterNodeでWorkerNodeにラベル付けを行う
    ```sh
    kubectl label node rpi-k8s-worker-01 node-role.kubernetes.io/worker=worker
    kubectl label node rpi-k8s-worker-02 node-role.kubernetes.io/worker=worker
    ```
* クラスタ全体を確認して完成
    ```sh
    kubectl get nodes
    ```
    ```plain
    NAME                STATUS   ROLES    AGE     VERSION
    rpi-k8s-master-01   Ready    master   8m34s   v1.14.1
    rpi-k8s-worker-01   Ready    worker   84s     v1.14.1
    rpi-k8s-worker-02   Ready    worker   66s     v1.14.1
    ```
