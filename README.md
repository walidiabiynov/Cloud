.
├── app

|   ├── __init__.py

│   ├── crud.py

│   ├── database.py

│   ├── main.py

│   ├── models.py

│   ├── routers

│   │   
    ├── __init__.py
│   │   
    ├── players.py
│   │   
    ├── seasons.py
│   │   
    └── teams.py
│   └── schemas.py

├── .pre-commit.yaml

├── LICENSE

├── README.md

├── requirements.txt

└── template.yml


crud.py: spécifie les actions crud (créer, lire, mettre à jour, supprimer)
database.py: configure la connexion avec PostgreSQL
main.py: rassemble toutes les routes
models.py: modèles sqlalchemy spécifiés
schemas.py: modèles pydantic spécifiés, qui je crois dictent le format de sortie lorsque l'API est appelée
routers/: dossier contenant des sous-ensembles de routes
.pre-commit.yaml: fichier de configuration pour l'outil pre-commit
requirements.txt: exigences à installer lors de la construction du projet en utilisant sam
template.yml: essentiellement la recette pour déployer le projet sur AWS
Configuration (Linux)
Installer et configurer AWS CLI et SAM
Pour procéder à la configuration et au déploiement, AWS CLI et SAM doivent être installés et configurés sur votre machine.

Installer CLI
Configurer CLI
Installer SAM
Créer un rôle AWS
Console IAM >> Rôles >> Créer
Sélectionnez Service AWS comme type et choisissez Lambda et cas d'utilisation
Ajoutez des politiques :
AWSLambdaBasicExecutionRole : Autorisation d'envoyer des journaux vers CloudWatch
AWSLambdaVPCAccessExecutionRole : Autorisation de connecter notre fonction Lambda à un VPC
Terminez la création du rôle et définissez le nom comme fastapilambdarole. Ce nom correspond au rôle spécifié dans template.yml.
Créer un compartiment S3
Lorsque nous déployons notre code avec AWS SAM, un dossier zip de notre code sera téléchargé dans S3. Il existe deux options pour créer un compartiment S3.

(1) Dans la console AWS
(2) Avec le AWS CLI

lua
 
aws s3api create-bucket \
--bucket {votre nom de compartiment ici} \
--region eu-central-1 \
--create-bucket-configuration LocationConstraint=eu-central-1
Veuillez noter que les noms de compartiment S3 doivent être globalement uniques. Ainsi, le nom du compartiment que vous créez ici déterminera le nom du compartiment utilisé dans les étapes ultérieures. De plus, assurez-vous de changer la région à votre région locale.

Cloner le projet et tester localement
 
 
git clone https://github.com/KurtKline/fastapi-postgres-aws-lambda.git
cd fastapi-postgres-aws-lambda
# créer et activer un environnement virtuel
pip install -r requirements.txt
pip install uvicorn
Pour tester localement sans erreurs, PostgreSQL doit être installé sur votre machine locale, et les données d'exemple doivent être chargées dans une table de base de données.
Installation de PostgreSQL sur Ubuntu 20.04

Depuis le terminal Linux :

psql
postgres=# CREATE DATABASE fitness;
postgres=# CREATE TABLE fit (id serial, player varchar(50), team varchar(50), season varchar(50), data_date date, points float);
postgres=# \copy fit(player, team, season, data_date, points) from 'clean_fit.csv' with DELIMITER ',' CSV HEADER;
Démarrer FastAPI

 
 
uvicorn app.main:app --reload
# cliquez sur le lien pour ouvrir le navigateur à l'adresse http://127.0.0.1:8000
Une fois que vous avez cliqué sur le lien, ajoutez /docs ou /redoc à l'URL http://127.0.0.1:8000/docs. Vous verrez alors l'interface Swagger.

Configuration de l'instance RDS PostgreSQL
Pour déployer sur AWS, notre code ET notre base de données doivent être sur AWS. Voici quelques directives de base pour configurer l'instance RDS PostgreSQL.

Dans les paramètres de l'instance RDS, assurez-vous que Accessibilité publique est définie sur Oui
Spécifiez nom de la base de données initiale, qui sera utilisé dans pg_restore ci-dessous
Liste blanche IP
Créez un nouveau groupe de sécurité EC2
Règles entrantes : Type: Trafic total, Source: Mon IP
Ajoutez ce groupe de sécurité à l'instance RDS
Comment accéder à l'instance RDS depuis le terminal Linux :

css
 
psql \
   --host=<point de terminaison de l'instance DB à partir d'AWS> \
   --port=<port> \
   --username=<nom d'utilisateur principal> \
   --password \
   --dbname=<nom de la base de données>
Une fois les données chargées dans votre instance RDS PostgreSQL, vous pouvez définir Accessibilité publique sur Non si vous le souhaitez. Cela empêche simplement les sources externes, comme votre PC local, d'accéder à votre instance RDS.

Chargement de données dans RDS PostgreSQL
Voici deux options pour charger les données dans RDS PostgreSQL

Sauvegarde et restauration : si les données existent déjà dans PostgreSQL local
Sauvegarde (faite à partir de la ligne de commande) : $ pg_dump -Fc mydb > db.dump
Restaurer avec : pg_restore -v -h [point de terminaison RDS] -U [nom d'utilisateur principal ("postgres" par défaut)] -d [nom de la base de données RDS] [fichier de sauvegarde].dump
Vérifiez que le chargement a réussi en vous connectant avec le bloc psql indiqué ci-dessus
À partir du fichier .csv
D'abord, connectez-vous à RDS via psql comme indiqué ci-dessus. Selon le nom de votre base de données, fit=> affiché ci-dessous peut être différent pour vous.

sql
 
fit=> create table fit (id serial, player varchar(50), team varchar(50), season varchar(50), data_date date, points float);
fit=> \copy fit(player, team, season, data_date, points) from 'clean_fit.csv' with DELIMITER ',' CSV HEADER;
COPY 105
Plus d'options pour charger des données dans PostgreSQL RDS
https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL.Procedural.Importing.html

Déploiement du projet avec AWS SAM
Le fichier template.yml est utilisé pour le déploiement avec AWS SAM.

(1) Remplacez les valeurs dans template.yml spécifiées comme {remplacer}.
(2) Décommentez # openapi_prefix="/prod" dans app/main.py. Cela permet un accès approprié à l'API lorsqu'elle est déployée.
(3) Exécutez les étapes suivantes pour SAM dans le terminal Linux

 
sam validate
css
 
sam build --debug
css
 
sam package --s3-bucket {votre nom de compartiment ici} --output-template-file out.yml --region eu-central-1
css
 
sam deploy --template-file out.yml --stack-name example-stack-name --region eu-central-1 --no-fail-on-empty-changeset --capabilities CAPABILITY_IAM
Obstacles rencontrés en cours de route
Accès à l'instance PostgreSQL RDS localement : Assurez-vous que Accessibilité publique est défini sur Oui, sinon vous obtiendrez une erreur de délai d'attente.

psycopg2-binary au lieu de psycopg2 : Pour une raison quelconque, AWS lambda ne fonctionne pas bien avec psycopg2, même si cela fonctionne localement

Lambda VPC : Lorsque ce projet est déployé tel quel, la connexion VPC est définie sur aucun. J'ai dû changer cela en VPC personnalisé et ajouter mon VPC par défaut et mon groupe de sécurité ici. Cela a depuis été ajouté directement dans le fichier template.yml.

openapi_prefix="/prod" : Cette valeur doit correspondre à StageName: prod dans template.yml. Le projet exemple que j'ai tiré avait Prod avec un P majuscule, ce qui ne chargerait pas correctement les /docs et /redoc une fois déployés.

Besoin d'ajouter la politique AWSLambdaVPCAccessExecutionRole au fastapilambdarole, sinon vous obtiendrez des erreurs lors de l'utilisation de la commande sam deploy.
