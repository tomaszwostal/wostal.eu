---
title: "GitOps w praktyce - Argo CD"
date: 2023-11-05T22:05:09+01:00
draft: false
toc: true
images:
tags:
  - gitops
  - cicd
  - kubernetes
  - argo
  - argo-cd
---

W tym artylule zamierzam przedstawić proces konfigurowania Argo CD na klastrze Kubernetes. Przy użyciu Helma, stworzymy aplikację do "zarządzania" aplikacjami oraz skonfigurujemy Argo CD tak, aby mogło aktualizować się samo.

Wszystkie pliki wymienione w tym poście na blogu są dostępne w repozytorium [Git](https://github.com/DevOps-Toys/devops-toys) na GitHubie.

Argo CD to "deklaratywne narzędzie ciągłego dostarczania GitOps dla Kubernetes". Może monitorować twoje repozytoria źródłowe i automatycznie wdrażać zmiany w twoim klastrze.

Kubernetes orkiestruje zadania związane z wdrażaniem i zarządzaniem kontenerami. Uruchamia moje kontenery, zastępuje je, gdy ulegną awarii, i skaluje moją usługę w węzłach obliczeniowych mojego klastra.

Kubernetes najlepiej sprawdza się jako część przepływu pracy ciągłego dostarczania. Uruchamianie automatycznych wdrożeń, gdy nowy kod zostanie scalony, zapewnia szybkie dotarcie zmian do mojego klastra po przejściu przez spójny łańcuch dostaw.

## Czym jest ArgoCD?

Argo CD to narzędzie GitOps służące do automatycznego synchronizowania klastra z pożądanym stanem zdefiniowanym w repozytorium Git. 
Każde obciążenie jest zdefiniowane deklaratywnie za pomocą manifestu zasobu w pliku YAML. 
Argo CD sprawdza, czy stan zdefiniowany w repozytorium Git odpowiada temu, co jest uruchomione na klastrze, i synchronizuje go, jeśli wykryje zmiany.

Na przykład, zamiast ręcznie wykonywać polecenia CLI do aktualizacji zasobów Kubernetes za pomocą *kubectl apply* lub *helm upgrade*, aktualizujemy pliki YAML, 
które następnie zatwierdzamy i wysyłamy do naszego repozytorium Git. Wszystko jest opisane w manifeście aplikacji. 
Argo CD okresowo sprawdza zasoby zdefiniowane w manifeście pod kątem zmian i automatycznie synchronizuje je z tymi, które są uruchomione na naszym klastrze.

Połączenie z klastrem, czy to z laptopa programisty, czy z systemu CI/CD, nie jest już potrzebne, ponieważ zmiany są teraz pobierane z repozytorium Git przez operatora Kubernetes działającego wewnątrz klastra.

## Push kontra Pull w CI/CD

Historycznie większość implementacji CI/CD polegała na zachowaniu typu **push**. Wymagało to połączenia klastra z platformą CI/CD, a następnie użycia narzędzi takich jak *kubectl* i *helm* w ramach pipeline’u do stosowania zmian w Kubernetes.

Argo to system CI/CD oparty na modelu **pull**. Działa wewnątrz twojego klastra Kubernetes i pobiera źródło z twoich repozytoriów. Argo następnie stosuje zmiany za ciebie, bez konieczności ręcznej konfiguracji pipeline’u.
Ten model jest bezpieczniejszy niż przepływy pracy typu push. Nie musisz eksponować serwera API twojego klastra ani przechowywać poświadczeń Kubernetes na platformie CI/CD. 
Skompromitowanie repozytorium źródłowego daje atakującemu dostęp tylko do kodu, a nie do kodu i drogi do twoich działających wdrożeń.

## Podstawowe pojęcia

Argo jest łatwe do nauki, gdy zrozumiesz jego podstawowe pojęcia. Oto kilka terminów, z którymi warto się zapoznać.

- **Kontroler Argo** – Kontroler Aplikacji Argo to komponent, który instalujesz w swoim klastrze. Implementuje wzorzec kontrolera Kubernetes do monitorowania twoich aplikacji i porównywania ich stanu z repozytoriami.
- **Aplikacja** – Aplikacja Argo to grupa zasobów Kubernetes, które razem wdrażają wszystkie niezbędne komponenty. Argo przechowuje szczegóły aplikacji w klastrze jako instancje dołączonej Custom Resource Definition (CRD).
- **Stan faktyczny** – Stan faktyczny to obecny stan twojej aplikacji w klastrze, taki jak liczba utworzonych Podów i obraz, który jest uruchomiony.
- **Stan docelowy** – Stan docelowy to wersja stanu, która jest deklarowana przez twoje repozytorium Git. Gdy repozytorium się zmienia, Argo podejmuje działania, które ewoluują stan faktyczny w stan docelowy.
- **Odświeżenie** – Odświeżenie następuje, gdy Argo pobiera stan docelowy z twojego repozytorium. Porównuje zmiany ze stanem faktycznym, ale niekoniecznie stosuje je na tym etapie.
- **Synchronizacja** – Synchronizacja to proces stosowania zmian odkrytych przez odświeżenie. Każda Synchronizacja przybliża klaster do stanu docelowego.

Więcej informacji o terminologii Argo i architekturze narzędzia znajdziesz w oficjalnej [dokumentacji](https://argo-cd.readthedocs.io/en/stable/).

## Wymagania

Aby przejść wszyskie kroki opisane w tym artykule, będziesz potrzebować następujących rzeczy:

- Klaster Kubernetes (polecam kind)
- kubectl
- Helm
- Publiczne repozytorium git (żeby Argo CD mogło pobrać manifesty)

Dla ułatwienia nasze manifesty aplikacji będą przechowywane w publicznym repozytorium Git.
Ja używam GitHuba, ale może to być dowolne publiczne repozytorium Git, a GitLab, Gitea itp. działają równie dobrze.

## Tworzenie własnego charta Helm

Użyjemy Helma, aby zainstalować Argo CD za pomocą charta utrzymywanego przez społeczność z argoproj/argo-helm. Projekt Argo nie dostarcza oficjalnego charta Helma.

Konkretnie, zamierzamy stworzyć "chart parasolowy" Helma. Jest to w zasadzie niestandardowy chart, który opakowuje inny. 
Wprowadza oryginalny chart jako zależność i zastępuje domyślne wartości. W naszym przypadku tworzymy wykres argo-cd, który opakowuje chart argo-cd utrzymywany przez społeczność.

Korzystając z tego podejścia, mamy większą elastyczność w przyszłości, możliwie dodając dodatkowe zasoby Kubernetes. Najczęstszym przypadkiem użycia tego jest dodanie sekretów (które mogą być szyfrowane za pomocą sops lub SealedSecrets) do naszej aplikacji. 
Na przykład, jeśli używamy webhooków z Argo CD, mamy możliwość bezpiecznego przechowywania adresu URL webhooka w secrecie.

Aby utworzyć chart parasolowy, tworzymy katalog w naszym repozytorium Git:

```shell
mkdir -p charts/argo-cd
```

Następnie umieszczamy w nim plik Chart.yaml:

charts/argo-cd/Chart.yaml

```yaml
apiVersion: v2
name: argo-cd
version: 1.0.0
dependencies:
  - name: argo-cd
    version: 5.50.1
    repository: https://argoproj.github.io/argo-helm
```

Wersja naszego niestandardowego charta na potrzeby tego artytułu nie ma znaczenia i może pozostać taka sama. Ważna jest natomiast wersja zależności (w naszym przypadku 5.50.1). 
Ważne jest, abyśmy włączyli chart argo-cd utrzymywany przez społeczność jako zależność. 
Następnie tworzymy plik values.yaml dla naszego wykresu:

charts/argo-cd/values.yaml

```yaml
argo-cd:
  nameOverride: argocd
  fullnameOveride: argocd
  dex:
    enabled: false
  notifications:
    enabled: false
  applicationSet:
    enabled: false
  server:
    extraArgs:
      - --insecure
```

Aby zastąpić wartości charta zależności, musimy umieścić je pod nazwą zależności. 
Ponieważ nasza zależność w Chart.yaml nazywa się argo-cd, musimy umieścić nasze wartości pod kluczem argo-cd:. 
Jeśli nazwa zależności to abcd, umieścilibyśmy wartości pod kluczem abcd:.

Wszystkie dostępne opcje dla wykresu Helma Argo CD można znaleźć w pliku **values.yaml**.

Korzystamy z dość minimalnej instalacji i wyłączamy komponenty, które na tę chwilę są zbędne. Zmiany to:

- Wyłącz komponent dex (integracja z zewnętrznymi dostawcami uwierzytelniania).
- Wyłącz kontroler powiadomień (powiadamia użytkowników o zmianach stanu aplikacji).
- Wyłącz kontroler ApplicationSet (automatyczne generowanie Aplikacji Argo CD).
- Uruchamiamy serwer z flagą **--insecure**, aby obsługiwać interfejs użytkownika przez HTTP.

Zanim zainstalujemy nasz chart, musimy wygenerować dla niego plik blokady. Podczas instalacji charta Argo CD sprawdza plik blokady pod kątem zależności i pobiera je. Brak pliku blokady spowoduje błąd.

```shell
helm repo add argo-cd https://argoproj.github.io/argo-helm
```

```shell
helm dependency update charts/argo-cd/
```

Spowoduje to utworzenie plików Chart.lock i charts/argo-cd-<version>.tgz. Plik .tgz jest wymagany tylko do początkowej instalacji z naszej lokalnej maszyny.
Później nie będzie do niczego potrzebny więc możemy dodać go do **.gitignore**.


```shell
echo "charts/**/charts" >> .gitignore
```

Nasz niestandardowy chart jest gotowy i może być przesłany do naszego publicznego repozytorium Git:

```shell
git add charts/argo-cd
git commit -m 'Add argo-cd chart'
git push
```

Następnym krokiem jest instalacja naszego charta.

```shell
helm upgrade --install argo-cd charts/argo-cd -n argocd --create-namespace
```

Po chwili wszystko powinno być wdrożone:

```shell
kubectl get pods -o wide -n argocd
NAME                                                       READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
argo-cd-argocd-application-controller-0                    0/1     Running   0          7s    10.244.2.8   kind-worker    <none>           <none>
argo-cd-argocd-applicationset-controller-d46d5f75b-mh94x   1/1     Running   0          7s    10.244.2.5   kind-worker    <none>           <none>
argo-cd-argocd-redis-667446b467-69btd                      1/1     Running   0          7s    10.244.1.4   kind-worker2   <none>           <none>
argo-cd-argocd-repo-server-7bc886f7c8-x9dzg                0/1     Running   0          7s    10.244.2.6   kind-worker    <none>           <none>
argo-cd-argocd-server-699fcb64f5-82z7c                     0/1     Running   0          7s    10.244.2.7   kind-worker    <none>           <none>
```

## Dostęp do interfejsu użytkownika Web UI

Domyślnie Helm nie instaluje ingressu. Aby uzyskać dostęp do interfejsu, musimy przekierować port do usługi argocd-server na porcie 443:

```shell
 kubectl port-forward svc/argo-cd-argocd-server 8080:443 -n argocd
```

Następnie możemy odwiedzić stronę http://localhost:8080, aby uzyskać do niej dostęp, co spowoduje wyświetlenie formularza logowania. 
Domyślną nazwą użytkownika jest admin. 
Hasło jest generowane automatycznie, możemy je uzyskać, używając:

```shell
kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" -n argocd | base64 -
```

Należy zauważyć, że niektóre shelle (takie jak Zsh) wyświetlają na końcu znak procenta. Nie jest on częścią hasła.

Po zalogowaniu się zobaczymy pusty interfejs Web UI.

![Argo CD empty UI](/argocd-p1/argocd-ui-empty.png)

W tym momencie aplikacje Argo CD mogą być dodawane poprzez interfejs Web UI lub wiersz poleceń (CLI), ale chcemy zarządzać wszystkim w sposób deklaratywny (infrastruktura jako kod). 
Oznacza to, że musimy napisać manifesty aplikacji w formacie YAML i umieścić je w repozytorium Git.

## Aplikacje i aplikacja główna (devops-app)

Zazwyczaj, kiedy chcemy dodać aplikację do Argo CD, musimy dodać zasób **Application** w naszym klastrze Kubernetes. 
Zasób ten musi określać, gdzie znajdują się manifesty dla naszej aplikacji. 
Manifesty mogą być plikami YAML, chartami Helm, Kustomize lub Jsonnet. 
W tym artykule skupimy się na tworzeniu aplikacji, które wykorzystują Helm.

Na przykład, jeśli chcielibyśmy wdrożyć Prometheus (co zrobimy później), napisalibyśmy dla niego manifest aplikacji w formacie YAML i umieścilibyśmy go w naszym repozytorium Git. 
Określałby on URL do charta Prometheusa i nadpisałby wartości, aby go dostosować. 
Następnie zastosowalibyśmy manifest i czekalibyśmy na utworzenie zasobów w klastrze.

Najłatwiejszym sposobem zastosowania manifestu jest użycie interfejsu wiersza poleceń **kubectl**. 
Jednakże jest to krok manualny, podatny na błędy, niebezpieczny, i musimy go powtarzać za każdym razem, gdy dodajemy lub aktualizujemy aplikacje. 
Argo CD oferuje lepszy sposób na zarządzanie aplikacjami. 
Możemy zautomatyzować dodawanie/aktualizowanie aplikacji, tworząc aplikację, która implementuje wzorzec **app of apps**.
W tym artykule nazawiemy to "aplikacją devops" (devops-app).

Aplikacja główna to chart Helm, który renderuje manifesty aplikacji. 
Początkowo musi być dodany ręcznie, ale potem możemy po prostu zarządzać manifestami aplikacji za pomocą Git, 
a one będą wdrażane automatycznie.

Aby pokazać, jak to działa bardziej szczegółowo, następnie utworzymy aplikację główną (devops-app).

## Tworzenie wykresu Helm aplikacji głównej (devops-app)

Aplikacja główna będzie wykresem Helm. Można również używać manifestów Kubernetes w formacie YAML, ale po dodaniu większej liczby aplikacji powstanie wiele powtarzającego się kodu (takiego jak klastr docelowy), którego można uniknąć, umieszczając wartości w pliku wartości wykresu. Innym interesującym rozwiązaniem są ApplicationSets, ale nie będziemy ich omawiać w tym samouczku.

Tworzymy wykres w tym samym publicznym repozytorium Git co wcześniej. Umieszczamy w nim plik Chart.yaml i (pusty) plik values.yaml:

```shell
mkdir -p charts/devops-app/templates
touch charts/devops-app/values.yaml
```

charts/devops-app/Chart.yaml:

```yaml
apiVersion: v2
name: devops-app
version: 1.0.0
```

Tworzymy manifest aplikacji dla naszej aplikacji głównej (devops-app) w charts/devops-app/templates/devops-app.yaml. 
Upewnij się, że zastąpisz repoURL adresem swojego publicznego repozytorium Git:

charts/root-app/templates/devops-app.yaml:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: devops-app
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/DevOps-Toys/devops-toys.git
    path: charts/devops-app
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      selfHeal: true
```

Powyższa aplikacja obserwuje nasz chart aplikacji głównej (pod charts/devops-app/) i jeśli zostaną wykryte zmiany,
synchronizuje (to znaczy, że renderuje chart i stosuje wynikowe manifesty w klastrze).

Skąd Argo CD wie, że nasza aplikacja to chart Helm? Szuka pliku Chart.yaml w ścieżce w repozytorium Git. 
To samo dotyczy Kustomize, który będzie używany, jeśli znajdzie plik kustomization.yaml w katalogu.

Argo CD nie użyje polecenia helm install do instalacji wykresów. 
Renderuje charta przy użyciu helm template, a następnie stosuje wynik przy użyciu kubectl. 
Oznacza to, że nie możemy uruchomić helm list na lokalnej maszynie, aby uzyskać listę wszystkich zainstalowanych wydań.

Wysyłamy pliki do naszego repozytorium Git:

```shell
git add charts/devops-app
git commit -m 'Add devops-app'
git push
```
A następnie stosujemy manifest w naszym klastrze Kubernetes. 
Pierwszy raz musimy to zrobić ręcznie, później pozwolimy Argo CD zarządzać aplikacją główną i synchronizować ją automatycznie:

```shell
helm template charts/devops-app/ | kubectl apply -f - -n argocd
```

W interfejsie Web UI możemy teraz zobaczyć, że aplikacja główna została utworzona.

![Argo CD Devops App](/argocd-p1/argocd-ui-devops-app.png)

## Samozarządzalne Argo CD

Wcześniej zainstalowaliśmy Argo CD ręcznie, używając polecenia **helm install** na naszym lokalnym komputerze. 
Oznacza to, że aktualizacje Argo CD, takie jak podnoszenie wersji lub zmiany w pliku values.yaml, 
wymagają od nas ponownego wykonania polecenia CLI Helm z lokalnej maszyny. 
Jest to powtarzalne, podatne na błędy i niezgodne z tym, jak instalujemy inne aplikacje w naszym klastrze.

Rozwiązaniem jest pozwolenie Argo CD na zarządzanie samym sobą. 
Bardziej konkretnie: pozwalamy kontrolerowi Argo CD śledzić zmiany w charcie helm argo-cd w naszym repozytorium 
(charts/argo-cd), renderować wykres Helm i stosować wynikowe manifesty. 
Jest to wykonane przy użyciu **kubectl** i jest asynchroniczne, więc jest bezpieczne, żeby automatycznie
zrestartować pody Argo CD po jego wykonaniu.

Aby to osiągnąć, musimy utworzyć manifest aplikacji, który wskazuje na nasz chart Argo CD. 
Użyjemy tej samej wersji i pliku *values.yaml*, co przy naszej poprzedniej ręcznej instalacji, 
więc początkowo nie zostaną wprowadzone żadne zmiany w zasobach w klastrze.

Manifest aplikacji wygląda tak:

**charts/devops-app/templates/argo-cd.yaml**:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-cd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/DevOps-Toys/devops-toys.git
    path: charts/argo-cd
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      selfHeal: true
```

Możemy teraz uaktualnić repozytorium Git. 
Nie musimy generować pliku Chart.lock, ponieważ nie mamy żadnych zależności:

```shell
git add charts/root-app/templates/argo-cd.yaml
git commit -m 'Add argo-cd application'
git push
```

W interfejsie Web UI powinniśmy teraz zobaczyć, że aplikacja devops-app jest OutOfSync,
a następnie zmienia się na Syncing. Jeśli nie zobaczymy zmian od razu, 
prawdopodobnie jest to spowodowane domyślnym tempem wykrywania zmian co 3 minuty. 
Możemy to przyspieszyć, klikając przycisk **Refresh** na aplikacji devops-app, co wyzwoli ręczną synchronizację.

Dla szybszego wykrywania zmian warto rozważyć ustawienie webhooków, 
które wywołają synchronizację natychmiast po wypchnięciu do repozytorium Git.

Przegląd interfejsu Web UI Argo CD po utworzeniu aplikacji Argo CD

![Argo CD Argo CD App](/argocd-p1/argocd-ui-argo-cd-app.png)



Gdy aplikacja Argo CD będzie zielona (zsynchronizowana), to wszystko. Możemy wprowadzać zmiany w naszej instalacji Argo CD tak samo, jak zmieniamy inne aplikacje: zmieniając pliki w repozytorium i wypychając je do naszego repozytorium Git.

Ostateczna struktura katalogów powinna wyglądać tak:

```text
└── charts
    ├── argo-cd
    │   ├── Chart.lock
    │   ├── Chart.yaml
    │   └── values.yaml
    └── devops-app
        ├── Chart.yaml
        ├── templates
        │   ├── argo-cd.yaml
        │   └── devops-app.yaml
        └── values.yaml
```

Jako ostatni krok możemy usunąć *secret*, który Helm tworzy dla każdej ręcznej instalacji:

```shell
kubectl delete secret -l owner=helm,name=argo-cd -n argocd
```




