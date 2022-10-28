Trainer Carsten Wieschiolek

# AWS CLI installieren

```
sudo apt-get install awscli
aws --version
aws configure # us-east-2 als Region verwenden!
aws iam get-user # wird der eingerichtete User angezeigt?
```

In `.bashrc` am Ende einfügen:

```
complete -C /usr/bin/aws_completer aws
```

# Terraform installieren

```
wget https://releases.hashicorp.com/terraform/0.12.17/terraform_0.12.17_linux_amd64.zip
unzip terraform_0.12.17_linux_amd64.zip
sudo mv terraform /usr/local/bin
terraform -install-autocomplete # macht Eintragungen in .bashrc
```

Syntax Highlighting für `vim`:

```
sudo apt-get install vim-pathogen
mkdir -p ~/.vim/bundle
cd ~/.vim/bundle
git clone https://github.com/hashivim/vim-terraform.git
```

In der 12er-Version gab es einige Änderungen, u.a. eine ganze Zahl neuer Schlüsselwörter.

# Terraform erstmalig anwenden

```
terraform init -> einmalig pro Projekt, Plugin für den Provider wird heruntergeladen.
terraform validate
terraform plan
terraform apply
terraform destroy
```

AMI IDs sind eindeutig je AWS-Region => AMI ID und Region müssen zusammenpassen.

Erstes Beispiel im Verzeichnis `intro`: Rechner ist nicht nutzbar, weil weder
Portfreischaltung noch Login-User angelegt wurden.

Mit `terraform apply` erzeugte Instanzen immer wieder mit `terraform destroy` löschen,
sonst Inkonsistenzen in den gespeicherten Zuständen!

[Terraform providers](https://www.terraform.io/docs/providers)

Alle Dateien im Projektverzeichnis, die auf `.tf` enden, werden interpretiert.
Terraform ist eine deklarative Sprache. Terraform ermittelt selbst die Abhängigkeiten,
die evtl. vorhanden sind, z.B. DNS-Namen für eine Instanz, die ebenfalls mit Terraform erzeugt wird.

# Variablen

Variablen müssen deklariert werden, idealerweise in einer separaten Datei. Wie die Datei heißt, ist egal.

Initialisierung von Variablen:
- `terraform.tfvars` (Datei muss so heißen, ansonsten auf der Kommandozeile anzugeben)
- Umgebungsvariablen `TF_VAR_<variablenname>` (mit export deklarieren)
- Interaktive Eingabe

`image_id = "${var.ami_id}"` -> bis v0.11 (ab 0.12 noch möglich), `image_id = var.ami_id` -> ab v0.12

# Schlüsselwörter

Verwendung: `<schlüsselwort> "<typ>" "<name>" { ... }`

Der `<name>` ist ein Terraform-Name, über diesen kann ein Terraform-Objekt später im Code referenziert werden.

- `variable`
- `provider` -> Angabe der Plugin-Version (`version`)
- `resource`
- `terraform` -> Angabe der erforderlichen Terraform-Version (`required_version`); erlaubt keine Variablen
- `data` -> Angabe von Daten z.B. aus Template

# Einfachen Webserver mit terraform erzeugen

`busybox`: sehr einfacher kleiner Webserver, Prozess, der auf verschiedene Protokolle antwortet (z.B. `busybox httpd`).

`user_data`: läuft unter `root`, hier werden minimale Post-Creation-Schritte ausgeführt, etwa Anlage eines
Users `ansible` mit SSH-Schlüssel.

`resource "aws_security_group" "instance" { ... }` -> `instance` ist der Terraform-Name dieser Ressource,
mit `instance.id` wird ihre ID an anderer Stelle verwendet.

`ingress`-Regel: `from_port` und `to_port` sind Grenzen eines Port Range.

Terraform speichert den Zustand, bei Änderungen wird nur der geänderte Teil ausgeführt - z.B. Änderung des Port Range.

Zugriff auf die hochgefahrene Instanz: in `main.tf` unter `resource "aws_instance" ...` die Zeile hinzufügen:

```
key_name               = "tftraining"
```

(`tftraining` zuvor erzeugter Schlüssel) -> analog zur Auswahl des Schlüssels in der Konsole

# Webserver-Cluster mit Autoscaling

Ressourcentyp `aws_launch_configuration` -> ähnlich zu `aws_instance`, zusätzliches Schlüsselwort
`lifecycle`. Hierarchie: Launch Config in Autoscaling Group, Verbindung mit `aws_lb_listener` über
`aws_lb_listener_rule` -> `aws_lb_target_group` (action). `health_check`-Direktive in `aws_lb_target_group`.
In diesem Beispiel: wenn health check des LB keine Antwort bekommt, sorgt die Autoscaling
Group für Austausch.

AWS verteilt automatisch über die AZ in der Region.

LB ist der älteste bei AWS verfügbare Load Balancer. Mittlerweile gibt es auch ELB (Elastic LB)
oder ALB (Application LB, optimiert für bestimmte Anwendungen wie Datenbanken).

Visualisierung:

```
sudo apt-get install graphviz
terraform graph | dot -Tpng > lb_with_asg.png
```

# Terraform State

Im Projektverzeichnis: `terraform.tfstate`, nur lesen, nicht ändern! Diese Dateien nicht in Git einchecken.

Anzeige: `terraform state list` -> `terraform state show <resourcename>`

Gemeinsames Arbeiten am Projekt:
- zentrale State-Ablage im S3-Bucket (Terraform-Abschnitt: `backend "s3"`)
- Jenkins-Buildserver
- Lock in Dynamo DB

S3-Bucketnamen müssen weltweit eindeutig sein! (Ähnlich DNS-Namen)

Zur Übung: mit `terraform apply` erst den S3-Bucket und die DynamoDB-Tabelle erzeugen
(`~/terraform-training/file-layout-example/global/s3`). Dann den Abschnitt `backend "s3"` aus `mysql/main.tf` kopieren
und in `main.tf` im ersten `webserver-cluster`-Beispiel eintragen.
(Unter `file-layout-example` liegt auch ein `webserver-cluster`-Verzeichnis, das hier nicht verwenden!)

Übungsbeispiel mit externen Daten:
- Anlage `mysql`
- Anlage `webserver-cluster` mit Daten von `mysql` -> bei Abfrage der Variablen den State Bucket
und den Pfadnamen der State-Datei darin angeben.

# Module

Ein Satz Konfigurationsdateien in einem Verzeichnis ist ein Modul. Provider nicht im Modul,
sondern von aufrufender Stelle definiert.

Aktivierung von Modulen vor `init` oder `plan` erforderlich: `terraform get [-update]`,
wiederholen bei Parameter- oder Code-Änderungen. `terraform` durchsucht die `.tf`-Dateien nach
`"module"` und holt die Module aus der `source`. Abgelegt werden sie in `.terraform`.

Es gibt immer mindestens ein Root-Modul: Satz `.tf`-Dateien im Arbeitsverzeichnis.

Rückgabewerte: über `output`-Block. Outputs liegen im Terraform State, Wdh. der Ausgabe mit `terraform output`.

# Schleifen und Kontrollstrukturen

hat Terraform eigentlich nicht: deklarative Sprache.

Mit `count` wird eine Schleife eingeleitet (0, 1, 2):

```
count = 3
name = "neo.${count.index}"
```

Schleife über Liste:

```
count = length(var.user_names)
name = element(var.user_names, count.index)

count = var.enable_autoscaling
```

(effektiv vom Typ boolsch) -> wenn 1, wird die nachfolgende Ressource erzeugt, wenn 0, nichts.

Ab Terraform 0.12 gibt es `for` und `for_each`. Alles (auch `count`) nur innerhalb einer Ressource verwendbar.
