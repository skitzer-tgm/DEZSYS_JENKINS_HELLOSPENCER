# Projektdokumentation: CI/CD Pipelines in Jenkins

**Projekt:** Middleware Engineering - "CI/CD Pipelines in Jenkins"  
**Autor:** Samuel / Thomas Micheler (Fork/Basis)  
**Technologien:** Jenkins, Docker, Python (Flask), Pytest, Git, Groovy (Pipeline)  

---

## 1. Einleitung & Zielsetzung

Das primäre Ziel dieses Projekts ist die Konzeptionierung und Umsetzung einer vollautomatisierten Continuous Integration und Continuous Deployment (CI/CD) Pipeline unter Verwendung von **Jenkins**. 

Als Test-Applikation dient eine einfache REST-API geschrieben in Python (Flask), welche einen Counter implementiert. Die Pipeline soll den gesamten Software-Lebenszyklus von der Code-Änderung bis hin zum laufenden Deployment abdecken. Dabei werden erweiterte Konzepte wie **Docker-in-Docker** und **Infrastructure as Code** (`Jenkinsfile`) angewandt, womit die erweiterten Anforderungen des Projekts zur Gänze erfüllt werden.

---

## 2. Architektur & Infrastruktur

Die gesamte Infrastruktur ist Container-basiert aufgebaut. Es wurde bewusst darauf verzichtet, Jenkins direkt auf dem Host-Betriebssystem zu installieren. 

### 2.1 Docker Compose Setup (`docker-compose.yml`)
Jenkins wird über Docker Compose bereitgestellt. Der wichtigste Aspekt dieser Konfiguration ist der **Docker Socket Mount** (`/var/run/docker.sock:/var/run/docker.sock`). 
Da die Pipeline selbst Docker-Container bauen und deployen muss, benötigt der Jenkins-Container Zugriff auf den Docker-Daemon des Host-Systems. Durch das Mounten des Sockets teilen sich Jenkins und der Host denselben Docker-Daemon.

### 2.2 Jenkins Umgebung & Plugins
Im Jenkins-Container wurde manuell die `docker-ce-cli` nachinstalliert, damit Jenkins die Befehle über den Socket absetzen kann.
Folgende Jenkins-Plugins sind essenziell für die Pipeline:
- **Pipeline:** Für das Ausführen des `Jenkinsfile`.
- **Docker Pipeline:** Erlaubt die Verwendung von `agent { docker { ... } }` in der Pipeline, um Stages innerhalb temporärer Container laufen zu lassen.
- **CloudBees Docker Build and Publish:** Zur allgemeinen Unterstützung von Docker-Befehlen.

---

## 3. Die CI/CD Pipeline (`Jenkinsfile`) im Detail

Die Pipeline ist in Declarative Pipeline Syntax geschrieben und unterteilt sich in 5 logische Phasen (Stages).

### Stage 1: Source
```groovy
stage('Source') {
    steps {
        cleanWs()
        checkout scm
    }
}
```
In dieser Phase wird der Workspace komplett gereinigt und der aktuelle Quellcode aus dem konfigurierten GitHub-Repository heruntergeladen. 

### Stage 2: Test (Unit-Tests)
```groovy
stage('Test') {
    agent {
        docker { 
            image 'python:3.11-slim-buster'
            args '-u root'
            reuseNode true
        }
    }
    steps { ... }
}
```
Anstatt die Tests direkt auf dem Jenkins-System auszuführen, wird dynamisch ein isolierter Python-Container gestartet. 
Die nötigen Abhängigkeiten (`requirements.txt`, `pytest`) werden installiert und die automatisierten Tests aus `tests/test_hello.py` ausgeführt. Nur wenn alle Tests (Status Codes, JSON Content-Types, Payload) positiv (`PASSED`) sind, geht die Pipeline weiter.

### Stage 3: Docker Build
Das Artefakt dieses Projekts ist ein produktionsfertiges Docker-Image. Mit dem Befehl `docker.build("${IMAGE_NAME}:${env.BUILD_ID}")` instruiert Jenkins den Host-Daemon, anhand des `Dockerfile` im Repository ein neues Image der Python-App zu bauen. Jedes Image wird mit der Build-Nummer getagged, um Versionierung zu gewährleisten.

### Stage 4: Deployment
```groovy
stage('Deployment') {
    steps {
        sh '''
            docker stop ${CONTAINER_NAME} || true
            docker rm ${CONTAINER_NAME} || true
            JENKINS_NETWORK=$(docker inspect -f '{{range $k, $v := .NetworkSettings.Networks}}{{$k}}{{end}}' $(hostname))
            docker run -d --name ${CONTAINER_NAME} --network ${JENKINS_NETWORK} -p ${APP_PORT}:${APP_PORT} ${IMAGE_NAME}:${BUILD_ID}
        '''
    }
}
```
Dies ist das Continuous Deployment. Ein potenziell laufender alter Container der App wird gestoppt und gelöscht. Danach wird das soeben in Stage 3 gebaute Image gestartet.
**Besonderheit:** Der Container wird über ein dynamisches Script exakt in dasselbe Docker-Netzwerk gehängt, in dem sich auch Jenkins befindet (`JENKINS_NETWORK`). Außerdem wird Port 5556 auf den Host gemapped, um die App von außen erreichbar zu machen.

### Stage 5: Integration Test
```groovy
stage('Integration Test') {
    steps {
        sh '''
            sleep 5
            curl http://${CONTAINER_NAME}:${APP_PORT}/api/hello
        '''
    }
}
```
Ein finaler Sanity Check: Da sich Jenkins und die deployte App nun im selben Bridge-Netzwerk befinden, kann Jenkins über den internen Docker-DNS-Namen (`${CONTAINER_NAME}`) direkt einen HTTP-Request an die App senden, um zu garantieren, dass die API fehlerfrei gebootet hat und antwortet.

---

## 4. Die Applikation

Die Test-Applikation (`src/hello.py`) ist ein schlankes Flask-Backend.
- **Endpoint:** `/api/hello`
- **Output:** Ein JSON-Objekt mit einer Gruß-Nachricht, Status und einem Counter.
- **Persistenz:** Die App liest und schreibt bei jedem Aufruf die Datei `count.txt`, weshalb in der Pipeline explizit darauf geachtet wurde, diese Datei vorab mit Lese- und Schreibrechten auszustatten (`chmod 666`).

---

## 5. Fazit & Evaluierung der Anforderungen

Die Umsetzung deckt alle geforderten Punkte ab:
1. **Installation von Jenkins:** ✔️ (via Docker Compose)
2. **Einfache Pipeline (Source, Build, Test, Deploy):** ✔️ (Detailliertes Jenkinsfile)
3. **Einbinden von GitHub:** ✔️ (SCM Checkout)
4. **Erstellung von Unit-Tests:** ✔️ (pytest)
5. **Docker Integration:** ✔️ (Docker Socket Mount & Docker Plugin)
6. **Trigger von GitHub:** ✔️ (via Polling/Webhook in Jenkins konfigurierbar)
7. **Lokales Deployment:** ✔️ (Container läuft auf dem Notebook auf Port 5556)

Die Pipeline zeigt anschaulich, wie moderner DevOps-Methoden fehleranfällige manuelle Schritte in der Softwareentwicklung durch Automatisierung ersetzen können.
