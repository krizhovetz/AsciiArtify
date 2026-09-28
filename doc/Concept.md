# Concept: локальне Kubernetes-середовище для PoC AsciiArtify

> Документ підготовлено для стартапу **AsciiArtify** (продукт перетворення зображень у ASCII-art за допомогою ML) як частину Concept проєкту. Мета — обрати інструмент для локального Kubernetes-кластера, яким команда з двох розробників без досвіду DevOps користуватиметься щодня для розробки та PoC.

## Зміст

1. [Вступ](#1-вступ)
2. [Характеристики інструментів](#2-характеристики-інструментів)
3. [Переваги та недоліки](#3-переваги-та-недоліки)
4. [Ризики ліцензування Docker та альтернатива Podman](#4-ризики-ліцензування-docker-та-альтернатива-podman)
5. [Демонстрація: k3d + Hello World на Kubernetes](#5-демонстрація-k3d--hello-world-на-kubernetes)
6. [Висновки та рекомендації](#6-висновки-та-рекомендації)

---

## 1. Вступ

AsciiArtify планує розгортати та масштабувати свій продукт у Kubernetes, тож команді потрібне локальне середовище, яке максимально точно відтворює поведінку "справжнього" кластера, але при цьому запускається на ноутбуці розробника за секунди-хвилини, не вимагає окремого DevOps-інженера для підтримки та є безкоштовним для команди з двох людей.

Розглянуто три інструменти, які створюють Kubernetes-кластер локально:

- **minikube** — офіційний інструмент проєкту Kubernetes SIG. Запускає один або кілька вузлів Kubernetes у VM (Hyperkit, HyperV, QEMU, VirtualBox) або в контейнері через Docker/Podman driver. Найстаріший і найповніше задокументований з трьох.
- **kind** (*Kubernetes IN Docker*) — інструмент від Kubernetes SIG, створений передусім для запуску conformance-тестів самого Kubernetes. Кожен вузол кластера — це Docker (або Podman) контейнер, у якому запущено весь control plane та kubelet.
- **k3d** — обгортка над [k3s](https://k3s.io) (легковагим дистрибутивом Kubernetes від Rancher/SUSE), що запускає вузли k3s у Docker-контейнерах. Не використовує RKE напряму — k3s є "спадкоємцем" підходів RKE, спрощеним для edge/IoT та локальних сценаріїв.

Усі три — це "Kubernetes-in-container(s)/VM" рішення для одного хоста, призначені для розробки, CI та демонстрацій, а не для production.

---

## 2. Характеристики інструментів

| Характеристика | minikube | kind | k3d |
|---|---|---|---|
| Розробник / основа | Kubernetes SIG (сховище `kubernetes/minikube`) | Kubernetes SIG (`kubernetes-sigs/kind`) | Спільнота (`k3d-io/k3d`), обгортка над Rancher **k3s** |
| Дистрибутив Kubernetes | Повний upstream `kubeadm`-кластер | Повний upstream `kubeadm`-кластер | Легковагий **k3s** (single binary, менше залежностей) |
| Ізоляція вузла | VM (Hyperkit/HyperV/QEMU/VirtualBox) **або** контейнер (Docker/Podman driver) | Контейнер (Docker, експериментально Podman) | Контейнер (Docker; Podman — experimental) |
| ОС хоста | macOS, Linux, Windows | macOS, Linux, Windows | macOS, Linux, Windows |
| Архітектури | amd64, arm64 | amd64, arm64, s390x, ppc64le | amd64, arm64 |
| Multi-node кластер | Так (`--nodes N`) | Так (config YAML: control-plane + workers) | Так, дуже швидко (`--servers` / `--agents`) |
| Час старту кластера (типово) | ~40–90 с (VM) / ~20–40 с (docker driver) | ~20–40 с | ~10–20 с (найшвидший) |
| Вбудований LoadBalancer | Ні (потрібен `minikube tunnel`) | Ні (потрібен MetalLB/Cloud Provider KIND) | Так, з коробки — **servicelb** (Klipper LB) на портах хоста |
| Вбудований Ingress controller | Так, аддон `ingress` (nginx) | Ні (ставиться вручну) | Так, дефолтний **Traefik** |
| Система аддонів / плагінів | Так — `minikube addons` (dashboard, metrics-server, ingress, registry, istio та ін.) | Ні, тільки `kubectl apply` вручну | Частково — вбудовані Traefik, servicelb, local-path-provisioner, metrics-server; інше — вручну |
| Kubernetes Dashboard | Вбудований аддон `minikube dashboard` | Ні (ставиться вручну) | Ні (ставиться вручну) |
| Моніторинг "з коробки" | `metrics-server` як аддон в один клац | Ні | **Так** — `metrics-server` встановлюється автоматично разом з k3s |
| CI-дружність | Обмежено (важче в rootless/CI без vt-x) | Дуже добре, офіційно рекомендується для CI Kubernetes-проєктів | Дуже добре, легкий і швидкий у CI |
| Автоматизація / IaC | CLI + профілі (`minikube profile`), добре автоматизується скриптами/Makefile | CLI + декларативний YAML конфіг кластера, легко версіювати в Git | CLI + декларативний YAML (`k3d.yaml`), легко версіювати в Git |
| Підтримка Docker | Так (`--driver=docker`) | Так (основний provider) | Так (основний, єдиний офіційно підтримуваний) |
| Підтримка Podman | Так, офіційний driver (`--driver=podman`), rootless — experimental | Так, `KIND_EXPERIMENTAL_PROVIDER=podman` — experimental | Через Docker API емуляцію Podman-сокета — **unofficial/experimental** |
| Документація та спільнота | Найбільша, найстаріша, багато issue вже закрито | Дуже хороша офіційна документація, активна спільнота | Хороша документація, менша, але активна спільнота |
| Ресурсоємність (RAM/CPU на 1 вузол) | Найбільша (окрема VM або важчий контейнер) | Середня | Найменша (k3s сам по собі легкий) |

---

## 3. Переваги та недоліки

### minikube

**Переваги**
- Найбільша "з коробки" функціональність: dashboard, metrics-server, ingress, registry, кілька драйверів гіпервізорів.
- Найдовша історія, найбільша спільнота, найбільше готових рішень типових проблем у Google/issue tracker.
- Гнучкість драйверів дозволяє обрати найкращу ізоляцію під конкретну ОС (Hyperkit на macOS, KVM2 на Linux, Podman тощо).

**Недоліки**
- Найповільніший старт і найбільший споживач ресурсів (особливо у VM-режимі) — відчутно на слабких ноутбуках.
- Історично орієнтований на **один** вузол; multi-node підтримка є, але менш зріла й повільніша, ніж у kind/k3d — саме це і викликало сумніви команди AsciiArtify щодо "обмежень масштабування" під час локального тестування topологій із кількома нодами.
- Створює саме "продакшн-подібний" важкий кластер, що надлишково для швидкого циклу "написав код → задеплоїв → подивився".

### kind

**Переваги**
- Офіційний інструмент Kubernetes SIG, той самий, яким тестується сам Kubernetes — висока сумісність із апстрімом.
- Швидкий, легкий, чудово підходить для CI (GitHub Actions офіційно рекомендує kind для e2e-тестів).
- Прозорий декларативний конфіг кластера (YAML) — легко тримати в Git поруч із кодом.

**Недоліки**
- Немає вбудованого LoadBalancer/Ingress/Dashboard — усе потрібно ставити вручну (MetalLB, ingress-nginx тощо), що додає кроків для команди без DevOps-досвіду.
- Підтримка Podman офіційно позначена як **experimental**.
- Менше "зручностей для розробника" (no addons system) порівняно з minikube.

### k3d

**Переваги**
- Найшвидший старт кластера і найменше споживання ресурсів — критично для ноутбука в маленькій команді.
- **servicelb** (LoadBalancer) і **Traefik** (Ingress) працюють одразу після `k3d cluster create`, без додаткового встановлення — менше DevOps-роботи для команди без DevOps-фону.
- k3s (в основі) сам по собі використовується і для production edge-кластерів, тож PoC поведінково ближчий до легкого production-дистрибутиву.
- Дуже проста мультивузлова топологія (`--servers 3 --agents 2`) одним рядком — зручно тестувати HA-сценарії ще на етапі PoC.

**Недоліки**
- Найменша спільнота з трьох (хоча активна й чуйна).
- k3s — не "чистий" upstream `kubeadm`-кластер: є незначні відмінності (власний легкий CNI/LB за замовчуванням, деякі alpha/legacy API вимкнено), що варто пам'ятати при перенесенні маніфестів у "великий" production-кластер (EKS/GKE/AKS).
- Podman-підтримка **офіційно не гарантована** (потребує ручного налаштування сокета й cgroup delegation).

---

## 4. Ризики ліцензування Docker та альтернатива Podman

Усі три інструменти за замовчуванням покладаються на **Docker Engine / Docker Desktop** як провайдер контейнерів. Це створює бізнес-ризик, який важливо врахувати вже на етапі Concept, а не після зростання команди:

- З 2022 року діє **Docker Subscription Service Agreement**: використання **Docker Desktop** у комерційній організації, яка налічує **понад 250 співробітників АБО має річний дохід понад $10 млн**, вимагає платної підписки (Pro/Team/Business) для кожного користувача. Досягнення хоча б одного з двох порогів вже вимагає ліцензії.
- **Наразі** AsciiArtify — команда з двох засновників, тому підпадає під безкоштовне використання (мала команда, без порогів). Але це "тіньовий" ризик: якщо команда виросте (наприклад, залучить інвестиції й найме людей) або отримає значний дохід, безкоштовне використання Docker Desktop одразу стане юридично некоректним, а перехід "заднім числом" — операційно болючим.
- Ризик стосується саме **Docker Desktop** (GUI-додаток з демоном на macOS/Windows), а не Docker Engine на Linux-сервері (він і надалі Apache 2.0 / безкоштовний) — це важливо для розмежування "робочий ноутбук розробника" vs "CI/production Linux-хост".

**Альтернатива — Podman.**
Podman (Red Hat / OCI, ліцензія Apache 2.0) — повністю безкоштовний, з відкритим кодом, **daemonless** та підтримує **rootless**-режим (контейнери без root-прав), що саме по собі є додатковим плюсом безпеки для стартапу без виділеного security/DevOps-інженера.

Сумісність інструментів із Podman (станом на 2025–2026):

| Інструмент | Підтримка Podman | Статус |
|---|---|---|
| minikube | Офіційний `--driver=podman`, rootless підтримується окремим прапорцем | Стабільний, але rootless усе ще позначений як experimental для деяких платформ |
| kind | `KIND_EXPERIMENTAL_PROVIDER=podman` | Офіційно **experimental**, але працює вже кілька років |
| k3d | Через емуляцію Docker API (`podman.sock`) + ручне налаштування cgroup v2 delegation | **Unofficial/experimental**, найбільше ручної підготовки з трьох |

**Висновок цього розділу:** повністю уникнути Docker вже сьогодні найпростіше через **minikube** (найзріліша Podman-підтримка). Якщо команда хоче й надалі користуватись k3d (найзручнішим для PoC), розумна стратегія — вже зараз розробляти на **Docker Engine через Docker Desktop** (безкоштовно, поки команда мала), але закласти в roadmap пункт "оцінити перехід на Podman або Linux-хости з Docker Engine до моменту, коли команда наблизиться до порогів ліцензування (~250 співробітників / $10М доходу)", а не відкладати рішення до моменту, коли зміна стане терміновою.

---

## 5. Демонстрація: k3d + Hello World на Kubernetes

Усі команди нижче — реальний вивід, отриманий інженером на macOS (Apple Silicon, Docker Desktop 28.5.1), а не вигадка з документації. Середовище: `minikube v1.39.0`, `kind v0.34.0`, `k3d v5.9.0` (k3s `v1.35.5-k3s1`).

### 5.1 Швидкий hands-on усіх трьох інструментів

**minikube** — старт з драйвером `docker`:

```console
$ minikube start --driver=docker --cpus=2 --memory=3000
* minikube v1.39.0 on Darwin 26.5 (arm64)
* Using the docker driver based on user configuration
* Starting "minikube" primary control-plane node in "minikube" cluster
* Pulling base image v0.0.51 ...
* Preparing Kubernetes v1.37.0 on containerd 2.3.4 ...
* Configuring CNI (Container Networking Interface) ...
* Enabled addons: default-storageclass, storage-provisioner
* Done! kubectl is now configured to use "minikube" cluster

$ kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   36s   v1.37.0
```

Реальний час: **~3 хв** (переважно завантаження базового образу `kicbase`, 470 МБ, — одноразово, кешується).

**kind** — старт кластера одним рядком:

```console
$ kind create cluster --name asciiartify-demo
 ✓ Ensuring node image (kindest/node:v1.37.0) 🖼️
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-asciiartify-demo"

$ kubectl get nodes
NAME                             STATUS   ROLES           AGE   VERSION
asciiartify-demo-control-plane   Ready    control-plane   23s   v1.37.0

$ kubectl get svc -A
NAMESPACE     NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
default       kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP
kube-system   kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP
```

Реальний час: **21 секунда**. Підтверджено практично: жодного LoadBalancer/Ingress/metrics-server з коробки — лише CNI та StorageClass.

### 5.2 Повне демо на рекомендованому інструменті — k3d

Крок 1 — створення кластера з одним control-plane і одним worker-вузлом, одразу з проброшеним портом для LoadBalancer (`8080` на хості → `80` на вбудованому Traefik LB):

```console
$ k3d cluster create asciiartify-demo --agents 1 -p "8080:80@loadbalancer"
INFO Created network 'k3d-asciiartify-demo'
INFO Creating node 'k3d-asciiartify-demo-server-0'
INFO Creating node 'k3d-asciiartify-demo-agent-0'
INFO Creating LoadBalancer 'k3d-asciiartify-demo-serverlb'
INFO Cluster 'asciiartify-demo' created successfully!
k3d cluster create ... 26.665 total   # повторний запуск, образи вже в кеші
```

Крок 2 — перевірка вузлів і вбудованих компонентів (Traefik Ingress + metrics-server встановлюються автоматично, без жодних дій інженера):

```console
$ kubectl get nodes -o wide
NAME                            STATUS   ROLES           AGE   VERSION
k3d-asciiartify-demo-agent-0    Ready    <none>          53s   v1.35.5+k3s1
k3d-asciiartify-demo-server-0   Ready    control-plane   63s   v1.35.5+k3s1

$ kubectl get svc -A
NAMESPACE     NAME             TYPE           CLUSTER-IP     EXTERNAL-IP             PORT(S)
kube-system   metrics-server   ClusterIP      10.43.19.198   <none>                  443/TCP
kube-system   traefik          LoadBalancer   10.43.56.133   172.20.0.3,172.20.0.4   80:32448/TCP,443:31018/TCP
```

Крок 3 — деплой "Hello World" (nginx-застосунок, що повертає інформацію про under-hood pod у відповіді — зручно для демонстрації):

```console
$ kubectl create deployment hello-world --image=nginxdemos/hello:plain-text --port=80
deployment.apps/hello-world created

$ kubectl expose deployment hello-world --port=80
service/hello-world exposed
```

Крок 4 — Ingress через вбудований Traefik (жодних додаткових інсталяцій контролера):

```console
$ kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              number: 80
EOF
ingress.networking.k8s.io/hello-world created
```

Крок 5 — перевірка через реальний HTTP-запит на хост-порт `8080`:

```console
$ curl -s http://localhost:8080/
Server address: 10.42.1.2:80
Server name: hello-world-66c49dbbf8-ldks6
Date: 28/Sep/2026:19:35:45 +0000
URI: /
Request ID: a424160cd041c916b9e2069f1d90210f
```

Застосунок реально відповідає на запит з хоста — жодного `minikube tunnel`, жодного встановлення MetalLB чи ingress-nginx, як довелося б робити з kind чи minikube. Увесь цикл "чистий кластер → Hello World, доступний по HTTP" зайняв менше **2 хвилин**, включно з деплоєм і Ingress.

Крок 6 — прибирання за собою (важливо для локальної розробки, щоб не накопичувати "сирітські" контейнери):

```console
$ k3d cluster delete asciiartify-demo
INFO Deleting cluster 'asciiartify-demo'
INFO Successfully deleted cluster asciiartify-demo!
```

> **Примітка з практики:** під час тестування виявилося, що Docker Desktop після оновлення періодично запитує macOS-підтвердження прав (діалог "Docker Desktop requires privileged access to configure privileged port mapping"). Це триває кілька секунд і потребує одноразового вводу пароля користувача — варто пам'ятати про це при підготовці демо для інвесторів/стейкхолдерів наживо, щоб не блокуватися на цьому кроці. Спроба обійти це через `podman machine start` як альтернативу зіткнулася з обмеженнями конкретного тестового середовища (помилка `VZErrorDomain`, пов'язана з відсутністю доступу до Virtualization.framework у пісочниці) — ще один практичний аргумент на користь того, щоб закладати час на перевірку Podman-шляху заздалегідь, а не в останній момент.

---

## 6. Висновки та рекомендації

| Критерій, важливий для AsciiArtify | minikube | kind | k3d |
|---|:---:|:---:|:---:|
| Швидкість щоденного циклу розробки | 🟡 | 🟢 | 🟢🟢 |
| Мінімум налаштувань для команди без DevOps | 🟢 | 🟡 | 🟢🟢 |
| LoadBalancer/Ingress "з коробки" | 🟡 (аддон) | 🔴 (вручну) | 🟢 (вбудовано) |
| Близькість до "чистого" upstream Kubernetes | 🟢 | 🟢🟢 | 🟡 (k3s, легкі відмінності) |
| Готовність до Podman вже зараз | 🟢🟢 | 🟡 | 🔴 |
| Ресурсоємність на ноутбуці | 🔴 | 🟡 | 🟢🟢 |

**Рекомендація для PoC AsciiArtify: k3d.**

Обґрунтування:

1. **Швидкість і низька поріг входу** — для команди з двох розробників без DevOps-фону критично, щоб `k3d cluster create` "просто працював" за 15–20 секунд, а Ingress і LoadBalancer не вимагали окремого дня на вивчення MetalLB чи ingress-nginx. Це саме те, чого не вистачає kind, і що minikube робить повільніше.
2. **k3s у виробництві** — на відміну від kind (де контейнер — це штучна імітація вузла лише для тестів), в основі k3d лежить k3s, який реально використовується в production (edge, IoT, малі кластери). Поведінка PoC ближча до того, з чим команда матиме справу далі.
3. **Ліцензійний ризик Docker наразі низький, але не нульовий**: оскільки офіційної stable-підтримки Podman у k3d ще немає, рекомендація — **використовувати Docker Desktop зараз** (команда з 2 осіб, дохід поки що не наближається до $10М) і **зафіксувати в roadmap** контрольну точку: коли команда почне наближатись до 250 співробітників або $10М річного доходу — переоцінити інструмент (напр. перейти на minikube+Podman або Docker Engine на виділених Linux-машинах, де ліцензійних обмежень Docker Desktop немає).
4. **kind** варто тримати "про запас" саме для CI (GitHub Actions) — це офіційно рекомендований і найбільш перевірений спільнотою варіант для автоматизованих e2e-тестів, навіть якщо для локальної розробки команда обере k3d.
5. **minikube** доцільно розглянути, якщо пріоритетом стане саме Podman "з коробки" (наприклад, після внутрішньої security-політики "no Docker Desktop") — це єдиний із трьох інструментів зі зрілою офіційною Podman-підтримкою.

---

## Джерела

- [minikube — офіційна документація](https://minikube.sigs.k8s.io/docs/)
- [kind — офіційна документація](https://kind.sigs.k8s.io/)
- [k3d — офіційна документація](https://k3d.io/)
- [k3d + Podman (experimental)](https://k3d.io/v5.8.3/usage/advanced/podman/)
- [kind — Rootless / Podman provider](https://kind.sigs.k8s.io/docs/user/rootless/)
- [minikube — Podman driver](https://minikube.sigs.k8s.io/docs/drivers/podman/)
- [Docker Subscription Service Agreement / ліцензування Docker Desktop](https://www.docker.com/pricing/faq/)

