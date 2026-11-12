# Dokumentacja: Środowisko k3s (Kubernetes) dla modelu llamafile z bezprzerwową aktualizacją

Zadanie polega na zainstalowaniu lekkiej dystrybucji Kubernetesa (k3s), skonfigurowaniu wdrożenia (Deployment) i usługi (Service) dla silnika `llamafile` oraz przeprowadzaniu testów bezprzerwowej aktualizacji (Rolling Update), wycofywania błędnych zmian (Rollback) i ocenie zużycia zasobów.

## 1. Instalacja środowiska k3s i przygotowanie sekretu GHCR

Rozpoczęto od instalacji lekkiego klastra k3s na systemie Debian oraz skonfigurowania poświadczeń do pobierania obrazów kontenerowych z prywatnego rejestru GitHub Packages (GHCR).

* `sudo curl -sfL https://get.k3s.io | sh -`
  Pobranie i automatyczna instalacja k3s wraz z dodaniem usługi do `systemd`.

* `sudo chmod 644 /etc/rancher/k3s/k3s.yaml`
  Nadanie odpowiednich uprawnień do pliku konfiguracyjnego k3s, aby umożliwić pracę narzędzia `kubectl`.

* `sudo kubectl create secret docker-registry ghcr-secret --docker-server=ghcr.io --docker-username=<TWOJ_NICK_GITHUB> --docker-password=<TOKEN_PAT> --docker-email=<EMAIL>`
  Utworzenie sekretu w klastrze umożliwiającego Kubernetesowi autoryzowane pobieranie obrazów z organizacji GHCR.

## 2. Pliki konfiguracyjne środowiska Kubernetes

W katalogu `k8s/` utworzono manifesty definiujące architekturę wdrożeniową aplikacji.

### Plik `k8s/01-service.yaml` (Wystawienie usługi)

Usługa typu `NodePort` udostępnia aplikację na określonym porcie maszyny wirtualnej (30080), przekierowując ruch do kontenerów z etykietą `app: llm`.

```
apiVersion: v1
kind: Service
metadata:
  name: llm-service
spec:
  type: NodePort
  selector:
    app: llm
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30080

```

### Plik `k8s/02-deployment.yaml` (Wdrożenie i strategia aktualizacji)

Manifest opisujący uruchomienie kontenera. Definiuje strategię `RollingUpdate` z parametrem `maxUnavailable: 0`, politykę bezpieczeństwa (zarówno na poziomie Poda, jak i kontenera), limity zasobów, hak `preStop` z opóźnieniem wyłączenia oraz sondy stanu (`startupProbe`, `readinessProbe`) zapewniające płynną wymianę Podów bez przerw w działaniu.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm
  labels:
    app: llm
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: llm
  template:
    metadata:
      labels:
        app: llm
    spec:
      imagePullSecrets:
        - name: ghcr-secret
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
      containers:
        - name: llm
          image: ghcr.io/eternity-mlops/mlops-marcin-z/llamafile-engine:v1.1
          lifecycle:
            preStop:
              exec:
                command: ["sleep", "5"]
          imagePullPolicy: IfNotPresent
          args:
            - "--server"
            - "--host"
            - "0.0.0.0"
            - "-m"
            - "/models/qwen.gguf"
          envFrom:
            - configMapRef:
                name: llm-config
          ports:
            - containerPort: 8080
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              memory: "1024Mi"
              cpu: "1000m"
            limits:
              memory: "2048Mi"
              cpu: "2000m"
          startupProbe:
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 30
            periodSeconds: 2
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            periodSeconds: 2
          volumeMounts:
            - name: model-weights
              mountPath: /models
              readOnly: true
            - name: tmpfs-vol
              mountPath: /tmp
      volumes:
        - name: model-weights
          hostPath:
            path: /home/merlin/docker/models
            type: Directory
        - name: tmpfs-vol
          emptyDir:
            medium: Memory

```

## 3. Przygotowanie i publikacja nowej wersji obrazu (GHCR)

Utworzono nową etykietę (tag `v1.1`) dla zbudowanego wcześniej obrazu i wypchnięto ją do rejestru organizacji.

* `sudo docker login ghcr.io -u MerLinH-0`
  Zalogowanie demona Docker na koncie administratora do rejestru GitHub Packages z użyciem tokena Personal Access Token (PAT).

* `sudo docker tag ghcr.io/eternity-mlops/mlops-marcin-z/llamafile-engine:v1.0 ghcr.io/eternity-mlops/mlops-marcin-z/llamafile-engine:v1.1`
  Oznaczenie istniejącego obrazu nowym tagiem wersji `v1.1`.

* `sudo docker push ghcr.io/eternity-mlops/mlops-marcin-z/llamafile-engine:v1.1`
  Wypchnięcie nowej wersji obrazu do rejestru GHCR.

## 4. Test bezprzerwowej aktualizacji (Rolling Update)

Przeprowadzono aktualizację wdrożenia z wersji `v1.0` do `v1.1` przy jednoczesnym odpytywaniu usługi w stałej pętli testowej.

* **Pętla testująca dostępność HTTP:**

  ```
  while true; do curl --connect-timeout 1 -m 2 --retry 3 --retry-connrefused -s -o /dev/null -w "HTTP: %{http_code}\n" http://192.168.100.149:30080/health; sleep 1; done
  
  ```

* **Wywołanie aktualizacji obrazu:**

  ```
  sudo kubectl set image deploy/llm llm=ghcr.io/eternity-mlops/mlops-marcin-z/llamafile-engine:v1.1
  
  ```

* **Wynik:**
  W trakcie wymiany kontenerów w pętli odpytującej nie odnotowano ani jednego błędu połączenia (`000`) ani błędu serwera (`503`). Usługa zachowała 100% dostępności (`HTTP: 200`).

## 5. Sprawdzenie działania i odporności środowiska

### Sprawdzenie dostępności REST API z maszyny gospodarza

Wysłano zapytanie do punktu kontrolnego `/health` z zewnętrznej maszyny w celu weryfikacji poprawności routingu sieciowego NodePort.

```
curl http://192.168.100.149:30080/health

```

* **Odpowiedź usługi potwierdzająca prawidłowe działanie:**
  

### Test wycofania błędnego wdrożenia (Rollback)

Przetestowano zachowanie klastra w przypadku próby wdrożenia uszkodzonego lub nieistniejącego obrazu.

1. Wdrożono zepsuty tag:

   ```
   sudo kubectl set image deploy/llm llm=ghcr.io/eternity-mlops/mlops-marcin-z/llamafile-engine:0000
   
   ```

2. Obserwacja stanu: Nowy Pod zawiśnął w stanie `ErrImagePull`, ale stary Pod kontynuował pracę w stanie `Running`.

3. Cofnięcie wdrożenia:

   ```
   sudo kubectl rollout undo deploy/llm
   
   ```

* **Potwierdzenie prawidłowej reakcji klastra i powrotu do stabilnego stanu:**
  

### Test autostartu po restarcie maszyny wirtualnej

Przeprowadzono restart systemu operacyjnego w celu weryfikacji mechanizmów rezyliencji i samonaprawy k3s.

1. `sudo reboot` – twardy restart maszyny wirtualnej.

2. Po ponownym połączeniu SSH wykonano `sudo kubectl get pods`.

* **Weryfikacja automatycznego uruchomienia kontenerów bez interwencji ręcznej:**
  

## 6. Metryki zasobów i tabela porównawcza

Pobrano dane dotyczące zużycia procesora oraz pamięci operacyjnej za pomocą narzędzia `kubectl top`.

* `sudo kubectl top pods` – zużycie zasobów przez sam kontener z modelem.

* `sudo kubectl top nodes` – całkowite zużycie zasobów maszyny (uwzględniające narzut k3s).

### Tabela porównawcza: Docker Compose vs Kubernetes (k3s)

| Kryterium | Docker Compose (Pracownia 3) | Kubernetes k3s (Pracownia 4) | 
| ----- | ----- | ----- | 
| **Zużycie RAM usługi (Pod/Kontener)** | \~1.9 GB | 1969 MiB (\~1.92 GB) | 
| **Całkowite zużycie RAM systemu** | \~2.3 GB | 3378 MiB (\~3.3 GB) | 
| **Narzut zarządcy (Orkiestratora)** | Niski (\~100-150 MB daemon Docker) | Średni (\~1.1 GB dla procesów k3s/systemd) | 
| **Zużycie CPU w stanie spoczynku** | \~0% | 1m (Pod) / 125m (Cały węzeł k3s) | 
| **Bezprzerwowa aktualizacja (Rolling Update)** | Brak (wymaga ręcznego reloadu/down) | Wbudowana z natywną obsługą sond `readiness` | 
| **Mechanizm samonaprawy i Rollback** | Podstawowy (`restart: unless-stopped`) | Zaawansowany (`kubectl rollout undo`, detekcja awarii) | 
| **Złożoność konfiguracji** | Niska (pojedynczy plik `compose.yaml`) | Średnia/Wysoka (pliki Deployment, Service, Sekrety) | 
