---
title: "Wprowadzenie do Kind"
date: 2023-11-01T21:50:43+01:00
draft: true
toc: true
images:
tags: 
  - kind
  - k8s
  - kubernetes
  - devops
---
Obecnie Kubernetes jest liderem wśród narzędzi do orkiestracji kontenerów. Zastanawiałeś się kiedyś, jak zapoznać się z jego komponentami, komendami czy innymi powiązanymi rozwiązaniami?

Jeśli potrzebujesz platformy, na której mógłbyś eksperymentować z Kubernetesem, dobrą wiadomością jest to, że masz wiele narzędzi do wyboru. Kubeadm, Kops, Kubespray, Minikube, Rancher Desktop i Killercoda to tylko niektóre z dostępnych opcji. Niemniej jednak każda z nich ma swoje ograniczenia. Niektóre środowiska są tymczasowe, 
jak w przypadku Killercody, inne pozwalają na utworzenie jedynie pojedynczego węzła sterującego z jedną bazą danych etcd, jak kubeadm, a jeszcze inne oferują tylko jeden klaster z jednym węzłem, jak minikube. Często też musisz płacić za zużywane zasoby.

Co by było, gdyby istniała możliwość stworzenia lokalnego klastra Kubernetes o wysokiej dostępności do celów rozwojowych i testowych? Klaster, który byłby trwały i nie wymagał dodatkowych opłat? Brzmi zachęcająco, prawda? A co, gdyby konfiguracja takiego klastra była prostym zadaniem?

Mówimy tutaj o **kind** (kubernetes in docker) – narzędziu pozwalającym uruchamiać lokalne klastry Kubernetes w kontenerach Docker. Chociaż głównym celem kind jest testowanie Kubernetesa, jest ono doskonałym rozwiązaniem do lokalnego rozwoju oraz integracji ciągłej (CI).

W tym artykule przyjrzymy się bliżej kind. Poznasz sposób jego użycia do tworzenia klastrów jedno- i wielowęzłowych oraz dowiesz się, jak wdrażać aplikacje w klastrze kind.

## Czym jest kind?

Kind to narzędzie, które oferuje wiele unikalnych funkcji ułatwiających uruchamianie lokalnych klastrów Kubernetes. Jest to projekt wspierany przez Kubernetes SIGs i znacząco różni się od powszechnie używanego Minikube.
Główną jego cechą jest możliwość tworzenia klastra w kontenerach Docker, co przyspiesza jego uruchamianie w porównaniu z metodami opartymi na maszynach wirtualnych.

Dokumentacja kind jest przystępna i zrozumiała. Więcej informacji znajdziesz [tutaj](https://kind.sigs.k8s.io/).

## Instalacja

### Wymagania

* Docker: Jest niezbędny do działania kind. Jeśli go nie masz, możesz pobrać go [stąd](https://docs.docker.com/engine/install/).
* Podman: Alternatywa dla dockera, bardzo ciekawy projekt. Szczegóły znajdziesz [tutaj](https://podman.io/)
* Kubectl (opcjonalnie): kind nie wymaga kubectl, ale bez tego narzędzia nie będziesz mógł wykonać niektórych operacji opisanych w tym poradniku. Kubectl jest dostępny do pobrania [stąd](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).

### Instalacja kind

W zależności od systemu operacyjnego proces instalacji może się nieco różnić. Poniżej znajdują się ogólne instrukcje dla najpopularniejszych systemów.

* MacOS:
  
  * używając Homebrew:

    ```shell
    brew install kind
    ```

  * używając MacPorts:

    ```shell
    sudo port selfupdate && sudo port install kind
    ```

  * plików binarnych:

    ```shell
    # for Intel Macs
    [ $(uname -m) = x86_64 ]&& curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-darwin-amd64
    # for M1 / ARM Macs
    [ $(uname -m) = arm64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-darwin-arm64
    chmod +x ./kind
    mv ./kind /some-dir-in-your-PATH/kind
    ```

* Windows:
  
  * używając Chocolatey:

    ```shell
    choco install kind
    ```

  * używając pakietów binarnych:

    ```shell
    curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.17.0/kind-windows-amd64
    Move-Item .\kind-windows-amd64.exe c:\some-dir-in-your-PATH\kind.exe
    ```
* Linux:

  * używając pakietów binarnych:

    ```shell
    curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64
    chmod +x ./kind
    sudo mv ./kind /usr/local/bin/kind
    ```
  * używając Homebrew:

    ```shell
    brew install kind
    ```
  * Oczywiście większość dystrybucji Linuksa posiada kind w swoich repozytoriach i można po prostu użyć managara pakietów.

## Jak stworzyć klaster

### Klaster jednowęzłowy

Aby stworzyć klaster, wystarczy jedno polecenie:

```shell
kind create cluster
```

Kilkadziesiąt sekund później klaster jest gotowy do użycia, co można łatwo sprawdzić używając kubectl:

```shell
kubectl cluster-info
```

```shell
kubectl get nodes
```

### Klaster wielowęzłowy

Niewątpliwą zaletą kind jest możliwość uruchamiana wielowęzłowych klastrów. Żeby to osiągnąć wystarczy stworzyć plik konfiguracyjny **kind.yaml**:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: control-plane
- role: worker
- role: worker
```

Teraz wystarczy stworzyć klaster za pomocą polecenia:

```shell
kind create cluster --config kind.yaml
```

Chwilę później możemy cieszyć się klastrem z czterema węzłami (dwa zarządcze i dwa robocze).

## Konfiguracja klastra

Domyślnie konfiguracja dostępu do klastra znajduje się w pliku **~/.kube/config** o ile zmienna **$KUBECONFIG** nie jest ustawiona.

## Zmiana obrazu węzła

kind pozwala również na bardzo łatwą zmianę obrazu użytego do uruchomienia klastra. Pozwala to na łatwe żonglowanie wersjami Kubernetesa, co nie zawsze jest możliwe w rozwiązaniach chmurowych. Lista wspieranych wersji znajduje się [tutaj](https://github.com/kubernetes-sigs/kind/releases).

```shell
kind create cluster --image kindest/node:v1.21.14@sha256:8a4e9bb3f415d2bb81629ce33ef9c76ba514c14d707f9797a01e3216376ba093
```

Powyższe polecenie uruchomi nam klaster k8s w wersji 1.21.14. Spróbuj to zrobić dzisiaj w Azure :)


## Dynamic Volume Provisioning

Dynamic Volume Provisioning w Kubernetes to mechanizm pozwalający na tworzenie wolumenów pamięci masowej na żądanie.

Domyślnie Kind ma już wstępnie skonfigurowaną, gotową do użycia klasę pamięci, która jest dostępna w momencie tworzenia klastra.

Żeby wyświetlić dostępne klasy wystarczy wydać polecenie:

```shell
kubectl get sc
```

Stwórzmy zatem PVC używają poniższego kodu:

```yaml
# local path provisioner only supports readwriteonce
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

Zakładając, że nazwa pliku to **pvc.yaml**:

```shell
kubectl create -f pvc.yaml
```

Potrzebny będzie jeszcze jeden plik yaml dla poda busybox:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  volumes:
  - name: host-volume
    persistentVolumeClaim:
      claimName: pvc
  containers:
  - image: busybox
    name: busybox
    command: ["/bin/sh"]
    args: ["-c", "sleep 600"]
    volumeMounts:
    - name: host-volume
      mountPath: /data
```

Poniższa komenda utworzy poda w klastrze:

```shell
kubectl apply -f busybox.yaml
```

Żeby potwierdzić, że wszystko działa jak należy użyjemy następujących poleceń:

```shell
kubectl get pv,pvc
```

```shell
kubectl get pod
```

Mamy teraz wielowęzłowy klaster z zamontowanym wolumenem w podzie.

Po wdrożeniu usługi na Kubernetesie trzeba ją jeszcze udostępnić. Dostęp do klastra można uzyskać na trzy sposoby: ingress, loadbalancer lub node port.

## Wdrażanie aplikacji

Najprostszym i najszybszym sposobem na wdrożenie aplikacji jest użycie **kubectl**. Jako przykład posłuży **Nginx**. Bardzo podstawowy deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.3
        ports:
        - containerPort: 80
```

Wystarczy zapisać ten plik jako **deployment.yaml** i użyć poniższej komendy:

```shell
kubectl apply -f deployment.yaml
```

Zostanie utworzony deployment z trzema replikami serwera Nginx.

Żeby połączyć się z serwerem www potrzebny jest serwis, który udostępnu usługę. Poniżej znajduje się przykładowa definicja serwisu:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: ClusterIP
```

Zapisujemy plik pod nazwą **service.yaml** i uruchamiamy polecenie:

```shell
kubectl apply -f service.yaml
```

Jeśli wszystko poszło zgodnie z planem wystarczy jeszcze przekierować port:

```shell
kubectl port-forward svc/nginx-service 9090:80
```

I sprawdzić w przeglądarce czy wszystko działa. Stona powinna być dostępna pod [tym adresem](http://127.0.0.1:9090).

## Eksportowanie logów klastra

Kind posiada możliwośc wyeksportowania wszystkich logów. Jest to szczególnie ważne w przypadku użycia klastra w porcesach CI/CD.

Żeby wyeksportować logi wystarczy jedna komenda:

```shell
kind export logs
```

## Podsumowanie

Po lekturze tego artykułu dowiesz się jak zainstalować kind, stworzyć klaster i wdrożyć aplikację. Jest to oczywiście wierzchołek góry lodowej i temat ten będę rozwijał w kolejnych artykułach.
Kind jest bardzo użytecznym narzędziem nie tylko do testowania Kubernetesa samego w sobie, ale też różnych konfiguracji ale i szerokiej gamy narzędzi i aplikacji. 
