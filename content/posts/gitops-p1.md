---
title: "Czym jest GitOps"
date: 2023-11-05T09:10:44+01:00
draft: true
toc: true
images:
tags: 
  - gitops
---

GitOps to sposób implementacji ciągłego wdrażania dla aplikacji typu cloud native.
Skupia się na doświadczeniu skierowanym do programistów podczas operowania infrastrukturą, korzystając z narzędzi, które są im już dobrze znane, takich jak Git i narzędzia do ciągłego wdrażania.

Główna idea GitOps polega na posiadaniu repozytorium Git, które zawsze zawiera deklaratywne opisy infrastruktury obecnie pożądanej w środowisku produkcyjnym oraz zautomatyzowanego procesu dostosowywania środowiska produkcyjnego do stanu opisanego w repozytorium. 
Jeśli chcesz wdrożyć nową aplikację lub zaktualizować istniejącą, wystarczy zaktualizować repozytorium - zautomatyzowany proces zajmuje się wszystkim innym. To tak, jakby mieć tempomat do zarządzania aplikacjami w produkcji.

## Dlaczego powinno się używać GitOps

### Szybkie i częste wdrażanie

By być sprawiedliwym, prawdopodobnie każda technologia ciągłego wdrażania pozwala na szybkie i częste wdrażanie aplikacji. 
Jednak unikalnym aspektem GitOps jest to, że nie musisz zmieniać narzędzi do wdrażania aplikacji.
Wszystko dzieje się w systemie kontroli wersji, którego i tak używasz do tworzenia aplikacji.

Kiedy mówię o **wysokiej prędkości**, mam na myśli to, że każdy zespół produktowy może bezpiecznie dostarczać aktualizacje wiele razy dziennie — wdrażać natychmiast, obserwować wyniki w czasie rzeczywistym i wykorzystywać tę informację zwrotną do kontynuowania lub cofania zmian.

### Łatwe i szybkie odzyskiwanie po błędach

O nie! Twoje środowisko produkcyjne przestało działać! Z GitOps masz kompletną historię zmian w twoim środowisku na przestrzeni czasu. 
To sprawia, że odzyskiwanie po błędach jest tak proste jak wydanie polecenia git revert i obserwowanie, jak twoje środowisko jest przywracane.

Repozytorium Git staje się wtedy nie tylko dziennikiem audytu, ale również dziennikiem transakcji. Możesz cofać się i przenosić do dowolnego zapisanego stanu.

### Łatwiejsze zarządzanie poświadczeniami

GitOps pozwala zarządzać wdrożeniami całkowicie z wnętrza twojego środowiska. Wystarczy, że twoje środowisko ma dostęp do twojego repozytorium i rejestru obrazów. To wszystko. Nie musisz dawać deweloperom bezpośredniego dostępu do środowiska.

Kubectl to nowe ssh. Ogranicz dostęp i używaj go do wdrażania tylko wtedy, gdy lepsze narzędzia nie są dostępne.

### Samodokumentujące się wdrożenia

Czy kiedykolwiek łączyłeś się z serwerem przez SSH i zastanawiałeś się, co na nim działa? Z GitOps każda zmiana w dowolnym środowisku musi nastąpić przez repozytorium. 
Zawsze możesz sprawdzić gałąź main, aby uzyskać kompletny opis tego, co jest wdrożone gdzie, plus pełną historię każdej dokonanej zmiany w systemie. I za darmo otrzymujesz ślad audytowy wszelkich zmian w systemie!

### Współdzielenie wiedzy w zespołach

Używanie Gita do przechowywania pełnych opisów wdrożonej infrastruktury pozwala każdemu w zespole śledzić jej ewolucję w czasie.
Dzięki świetnym wiadomościom w commitach każdy może odtworzyć proces myślowy zmian infrastruktury, a także łatwo znaleźć przykłady ustawień nowych systemów.

GitOps to najlepsza rzecz od czasu konfiguracji jako kod. Git zmienił sposób w jaki współpracujemy, ale deklaratywna konfiguracja jest kluczem do radzenia sobie z infrastrukturą na dużą skalę i otwiera drogę dla nowej generacji narzędzi zarządzania.

## Jak działa GitOps?

### Konfiguracje środowiskowe jako repozytorium Git

GitOps organizuje proces wdrażania wokół repozytoriów kodu jako centralnego elementu. Istnieją przynajmniej dwa repozytoria: repozytorium aplikacji i repozytorium konfiguracji środowiska.
Repozytorium aplikacji zawiera kod źródłowy aplikacji oraz manifesty wdrożeniowe do wdrażania aplikacji.
Repozytorium konfiguracji środowiska zawiera wszystkie manifesty wdrożeniowe obecnie pożądanej infrastruktury środowiska wdrożeniowego.
Opisuje, jakie aplikacje i usługi infrastrukturalne (broker wiadomości, siatka usług, narzędzie monitorujące, ...) powinny działać z jaką konfiguracją i wersją w środowisku wdrożeniowym.

### Wdrażanie oparte na mechanizmie push vs pull

Istnieją dwa sposoby implementacji strategii wdrażania dla GitOps: wdrażania oparte na mechanizmie **push** oraz **pull**. 
Różnica między tymi dwoma typami wdrażania polega na sposobie zapewnienia, że środowisko wdrożeniowe faktycznie odpowiada pożądanej infrastrukturze. 
Gdzie to możliwe, preferowane powinno być podejście oparte na mechanizmie **pull**, ponieważ jest ono uważane za bezpieczniejsze i tym samym lepszą praktykę wdrażania GitOps.

### Wdrażanie oparte na mechanizmie push

Strategia wdrażania oparta na mechanizmie **push** jest implementowana przez popularne narzędzia CI/CD, takie jak Jenkins, CircleCI czy Travis CI.
Kod źródłowy aplikacji znajduje się w repozytorium aplikacji wraz z wymaganymi plikami YAML Kubernetes, potrzebnymi do wdrożenia aplikacji.
Kiedykolwiek kod aplikacji jest aktualizowany, uruchamiany jest łańcuch budowania, który buduje obrazy kontenerów, a ostatecznie repozytorium konfiguracji środowiska jest aktualizowane o nowe deskryptory wdrożenia.

Wskazówka: Możesz również przechowywać szablony plików YAML w repozytorium aplikacji. Gdy zbudowana jest nowa wersja, szablon może być użyty do wygenerowania pliku YAML w repozytorium konfiguracji środowiska.

![GitOps Push](/gitops-p1/gitops-push.png "https://gitops.tech/")

Zmiany w repozytorium konfiguracji środowiska inicjują łańcuch wdrożeniowy. Ten łańcuch jest odpowiedzialny za wdrożenie wszystkich manifestów z repozytorium konfiguracji środowiska do infrastruktury.
W tym podejściu niezbędne jest dostarczenie poświadczeń do środowiska wdrożeniowego. W związku z tym, łańcuch wdrożeniowy posiada uprawnienia typu "god-mode". 
W niektórych przypadkach użycie wdrożenia opartego na mechanizmie **push** jest nieuniknione, gdy uruchamiana jest automatyczna konfiguracja infrastruktury chmurowej. 
W takich sytuacjach zdecydowanie zaleca się wykorzystanie systemu autoryzacji dostępnego u dostawcy chmury, który pozwala na bardziej restrykcyjne uprawnienia wdrożeniowe.

Inną ważną kwestią do zapamiętania przy użyciu tego podejścia jest to, że łańcuch wdrożeniowy jest uruchamiany tylko wtedy, gdy zmienia się repozytorium środowiska. 
Nie jest w stanie automatycznie zauważyć żadnych odchyleń między stanem środowiska a jego pożądanym stanem. 
Oznacza to, że potrzebny jest jakiś sposób monitorowania, aby można było interweniować, jeśli środowisko nie odpowiada temu, co jest opisane w repozytorium środowiska.

### Wdrożenia oparte na mechanizmie pull

Strategia wdrożenia oparta na mechanizmie **pull** wykorzystuje te same koncepcje, co wariant oparty na mechanizmie **push**, ale różni się sposobem działania łańcucha wdrożeniowego. 
Tradycyjne łańcuchy CI/CD są uruchamiane przez zewnętrzne zdarzenie, na przykład gdy nowy kod jest wysyłany do repozytorium aplikacji. 
W podejściu wdrożeniowym opartym na mechanizmie **pull**, wprowadzony zostaje operator. 
Przejmuje on rolę łańcucha wdrożeniowego poprzez ciągłe porównywanie pożądanego stanu w repozytorium środowiska z rzeczywistym stanem w wdrożonej infrastrukturze. 
Kiedykolwiek zauważone są różnice, operator aktualizuje infrastrukturę, aby odpowiadała repozytorium środowiska. Dodatkowo rejestr obrazów może być monitorowany w celu znalezienia nowych wersji obrazów do wdrożenia.


![GitOps Pull](/gitops-p1/gitops-pull.png "https://www.gitops.tech/")

Podobnie jak w przypadku wdrożeń opartych na mechanizmie **push**, ta wersja aktualizuje środowisko za każdym razem, gdy repozytorium środowiska ulega zmianie.
Jednakże, dzięki operatorowi, zmiany mogą być także zauważane w przeciwnym kierunku. Za każdym razem, gdy wdrożona infrastruktura zmienia się w sposób nieopisany w repozytorium środowiska, takie zmiany są cofane. 
Zapewnia to, że wszystkie zmiany są śledzone w repozytorium Git, uniemożliwiając bezpośrednie zmiany w klastrze.

Ta zmiana kierunku rozwiązuje problem wdrożeń opartych na mechanizmie **push**, gdzie środowisko jest aktualizowane tylko wtedy, gdy repozytorium środowiska jest aktualizowane.
Jednakże nie oznacza to, że można całkowicie zrezygnować z monitoringu. Większość operatorów obsługuje wysyłanie maili lub powiadomień Slack, 
jeśli z jakiegokolwiek powodu nie może doprowadzić środowiska do pożądanego stanu, na przykład jeśli nie może pobrać obrazu kontenera. 
Dodatkowo, prawdopodobnie powinno się skonfigurować monitoring samego operatora, ponieważ bez niego nie istnieje już zautomatyzowany proces wdrożeniowy.

Operator powinien zawsze znajdować się w tym samym środowisku lub klastrze co aplikacja do wdrożenia. 
Zapobiega to sytuacji, jak w podejściu opartym na mechanizmie **push**, gdzie poświadczenia do wdrożeń są znane przez łańcuch CI/CD. 
Gdy instancja faktycznie wdrażająca znajduje się w tym samym środowisku, żadne poświadczenia nie muszą być znane zewnętrznym usługom. 
Mechanizm autoryzacji wykorzystywanej platformy wdrożeniowej może być wykorzystany do ograniczenia uprawnień dotyczących wdrażania. 
Ma to ogromne znaczenie z punktu widzenia bezpieczeństwa. W przypadku używania Kubernetes, można wykorzystać konfiguracje RBAC i konta usług.

### Praca z wieloma środowiskami i aplikacjami

Oczywiście praca tylko z jednym repozytorium aplikacji i jednym środowiskiem nie jest realistyczna dla większości aplikacji. 
Kiedy używasz architektury mikroserwisów, prawdopodobnie będziesz chciał przechowywać każdą usługę w jej własnym repozytorium.

GitOps może także obsłużyć taki przypadek użycia. Zawsze możesz skonfigurować wiele łańcuchów budowania, które aktualizują repozytorium środowiska. 
Stamtąd regularny zautomatyzowany przepływ pracy GitOps jest uruchamiany i wdraża wszystkie części aplikacji.


![GitOps Multiple](/gitops-p1/gitops-multiple.png "https://gitops.tech/")

Zarządzanie wieloma środowiskami za pomocą GitOps można przeprowadzić, używając oddzielnych gałęzi w repozytorium środowiskowym. 
Możesz skonfigurować operatora lub łańcuch budowania tak, aby reagował na zmiany w jednej gałęzi poprzez wdrażanie w środowisku produkcyjnym, a w innej - aby wdrażać w środowisku stagingowym.
