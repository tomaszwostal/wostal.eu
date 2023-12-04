---
title: "Monitoring Argo CD"
date: 2023-12-04T18:19:03+01:00
draft: false
toc: true
images:
tags:
  - gitops
  - cicd
  - kubernetes
  - argo
  - argo-cd
  - grafana
  - prometheus
  - monitoring
---

W każdym środowisku ważne jest monitorowanie. Pomaga to śledzić wydajność aplikacji/narzędzi oraz optymalizować aplikację.
Podobnie, w rozbudowanym środowisku, gdzie liczne zespoły wdrażają zarówno monolity, jak i mikroserwisy do Kubernetesa, niemal pewne jest, że operacje nie zawsze będą przebiegać idealnie.
Argo CD dostarcza nam mnóstwa metryk, które umożliwiają ocenę utylizacji systemu, czy to poniżej, czy powyżej oczekiwań, oraz planowanie niezbędnych działań.

W tym artykule przyjrzymy się, jak monitorować Argo CD za pomocą Prometheusa.

Prometheus wyróżnia się jako jedno z najskuteczniejszych narzędzi do oceny aktualnego stanu systemu i szybkiego wykrywania potencjalnych problemów.

![Metrics flow](/argocd-p2-monitoring/flow.png)

## Czym jest kube-prometheus-stack

**kube-prometheus-stack** to zbiór komponentów, które ułatwiają obsługę instancji Prometheus w klastrze Kubernetes.
Jest zaprojektowany, by dostarczać proste i kompleksowe rozwiązania monitorujące dla klastrów.

Zawiera predefiniowany zestaw dashboardów dla Grafany i reguł dla Alertmanagera, połączonych z najlepszymi praktykami opracowanymi przez społeczność.
Składniki tego zestawu to:

**Prometheus**: Oprogramowanie open-source do monitorowania systemów i alertowania.

**Alertmanager**: Obsługuje alerty wysyłane przez aplikacje klienckie, takie jak serwer Prometheus, dbając o ich deduplikację, grupowanie i kierowanie do odpowiednich odbiorców.

**Node Exporter**: Eksporter Prometheus dla metryk sprzętowych i systemowych z możliwością podłączania kolektorów metryk.

**kube-state-metrics**: Usługa, która nasłuchuje serwera API Kubernetes i generuje metryki o stanie obiektów.

**Grafana**: Otwarta platforma do monitorowania i obserwacji, umożliwiająca zapytania, wizualizację, alertowanie i zrozumienie metryk.

**Prometheus** Operator: Tworzy, konfiguruje i zarządza klastrami Prometheus na Kubernetes.

## Instalacja

### Przygotowania

W celu utrzymania porządku w charcie dla **Argo CD** został dodany projekt [monitoring](https://github.com/DevOps-Toys/devops-toys/blob/main/charts/argo-cd/templates/project-cicd.yaml), żeby łatwiej było tym wszystkim zarządzać.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: monitoring
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Monitoring Applications
  sourceRepos:
    - "*"
  destinations:
    - namespace: "*"
      server: "*"
  clusterResourceWhitelist:
    - group: "*"
      kind: "*"
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
```

Z punktu widzenia bezpieczeństwa jest to katastrofa ponieważ pozwalamy temu projektowi na zmianę wszystkiego i wszędzie, ale na zabezpiecznie klastra
przyjdzie czas w przyszłości. W tej chwili interesują nas metryki.

### Instalacja CRD

Z powodu problemu z definicjami CRD operatora Prometheus, najlepiej jest wdrożyć CRD oddzielnie.
W tym celu wystarczy dodać definicję **prometheus-crds.yaml** w charcie devops-app (charts/devops-app/templates/prometheus-crds.yaml).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus-crds
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "3"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: monitoring
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: prometheus-operator-crds
    targetRevision: 7.0.0
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

Sama definicja aplikacji jest banalnie prosta. Z ciekawostek została dodana anotacja _argocd.argoproj.io/sync-wave: "3"_, ale o tym też będzie
mowa w kolejnych częściach.
Kolejną nowością jest _SeverSideApply=true_. Z doświadczenia wiem, że w przypadku tych chartów lepiej użyć tej opcji. Powoduje ona wygnerowanie
ostatecznego, gotowego patcha po stronie serwera a nie klienta. Ma to znaczenie w przypadku dużych chartów.

### Instalacja kube-prometheus-stack

Do wdrożenia Prometheusa użyjemy prawie domyślnej konfiguracji, ustawiając jedynie **label**, dzięki czemu bez problemu Prometheus będzie mógł pobierać
metryki z sewrisów. Dodatkowo dodajemy dashboard, który wyświetli metryki dla Argo CD. Dashboard pobieramy jest bezpośrednio z repozytorium Argo CD.
Podobnie jak w poprzednim punkcie poniższą zawrtość wklejamy do pliku **charts/devops-app/templates/prometheus-stack.yaml**. I to wszystko jeżeli chodzi
o Prometheusa.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "3"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: monitoring
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 52.1.0
    helm:
      skipCrds: true
      values: |-
        nameOverride: "monitoring"
        fullnameOverride: "monitoring"
        grafana:
          dashboardProviders:
            dashboardproviders.yaml:
              apiVersion: 1
              providers:
              - name: 'cicd'
                orgId: 1
                folder: ''
                type: file
                disableDeletion: false
                editable: true
                options:
                  path: /var/lib/grafana/dashboards/cicd
          dashboards:
            cicd:
              argo-cd:
                url: https://raw.githubusercontent.com/argoproj/argo-cd/master/examples/dashboard.json
        prometheus:
          enabled: true
          agentMode: false
          prometheusSpec:
            podMonitorSelector:
              matchLabels:
                release: prometheus
            serviceMonitorSelector:
              matchLabels:
                release: prometheus
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### Konfiguracja Argo CD

Pozostaje nam jedynie włączyć metryki w Argo CD. W tym celu edytujemy plik **charts/devops-app/templates/argo-cd.yaml**:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-cd
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: ci-cd
  source:
    repoURL: https://github.com/DevOps-Toys/devops-toys.git
    path: charts/argo-cd
    targetRevision: HEAD
    helm:
      values: |-
        argo-cd:
          global:
          controller:
            metrics:
              enabled: true
              serviceMonitor:
                enabled: true
                labels:
                  release: prometheus
                selector:
                  release: prometheus
          dex:
            enabled: false
            serviceMonitor:
              enabled: false
              labels:
                release: prometheus
              selector:
                release: prometheus
          redis:
            enabled: true
            exporter:
              enabled: true
            metrics:
              enabled: true
              serviceMonitor:
                enabled: true
                labels:
                  release: prometheus
                selector:
                  release: prometheus
          server:
            metrics:
              enabled: true
              serviceMonitor:
                enabled: true
                labels:
                  release: prometheus
                selector:
                  release: prometheus
          repoServer:
            metrics:
              enabled: true
              serviceMonitor:
                enabled: true
                labels:
                  release: prometheus
                selector:
                  release: prometheus
          applicationSet:
            enabled: true
            metrics:
              enabled: true
              serviceMonitor:
                enabled: true
                labels:
                  release: prometheus
                selector:
                  release: prometheus
          notifications:
            enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      selfHeal: true
```

Jak widać, każda z usług wchodząca w skład Argo CD musi mieć włączone metryki i skonfigurowany **serviceMonitor**. Konfiguracja polega głównie
na podaniu odpowiedniego selektora, takiego, żeby pokrywał się z tym, który ustawiliśmy wcześniej konfigurując Prometheusa.

Po zatwierdzeniu zmiań w repozytorium, Argo CD powinno automatycznie wykryć zmiany i wprowadzić zmiany w konfiguracji.

## Testy

### Prometheus

Testowanie rozpoczynamy od sprawdzenia czy Promethues jest w stanie odczytać metryki. W tym celu należy przekierować
port z serwisu Prometheusa na swój komputer:

```shell
kubectl port-forward svc/monitoring-prometheus 9090:9090 -n monitoring
```

Teraz wystarczy w przeglądarce wpisać adres: [http://localhost:9090/service-discovery?search=](http://localhost:9090/service-discovery?search=). Efekt powinien być
podobny do tego:

![Prometheus Service discovery](/argocd-p2-monitoring/prometheus-sd.png)

Jak widać Prometheus wykrył i jest w stanie pobrać metryki.

### Grafana

Podobnie jak w przypadku Prometheusa sprawdzanie Grafany należy zacząć od przekierowania portu.

```shell
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
```

Od teraz Grafana jest dostępna pod adresem _http://localhost:3000_. Domyślny użytkownik to **admin**, natomiast
hasło to **prom-operator**.

Dashboard znajduje się pod adresem: [http://localhost:3000/d/LCAgc9rWz/argocd?orgId=1](http://localhost:3000/d/LCAgc9rWz/argocd?orgId=1)

![Grafana Argo CD](/argocd-p2-monitoring/grafana.png)

## Podsumowanie

Jak widać włączenie metryk nie jest zadaniem karkołomnym, należy jednak pamiętać, że przedstawiony tutaj przykład jest przykładem bardzo prostym
i pod żadnym pozorem nie powinno się go używać w środowiskach choćby przypominających produkcję. Do tego droga jest jeszcze bardzo daleka, ale myślę, że
z czasem dojdziemy i do tego etapu.
