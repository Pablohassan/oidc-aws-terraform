OIDC-AWS-Terraform Integration

Introduction
Ce projet démontre l'intégration d'un flux de travail de type CI/CD utilisant GitHub Actions avec AWS via Terraform. L'objectif est de créer une infrastructure sécurisée et efficace, permettant une gestion automatisée des ressources AWS.

Fonctionnement
Résumé : 
ce workflow configure l'environnement, vérifie le code Terraform, puis applique les changements sur AWS si tout est conforme.
Lorsqu'une Pull Request est fusionnée sur la branche principale, cela déclenche un workflow GitHub Actions qui exécute un apply dans AWS. 
Cette automatisation assure une cohérence et une fiabilité dans le déploiement de nos configurations.

Details du Github-actions: 

Déclencheurs:

Le workflow se déclenche sur des événements push sur la branche main et sur tous les pull_request.
Permissions:

id-token: write : Nécessaire pour la connexion OIDC avec AWS.
contents: read : Requis pour l'action checkout.
pull-requests: write : Permet au bot GitHub de commenter les Pull Requests.
Variables d'Environnement Globales:

TF_LOG : Niveau de log pour Terraform.
AWS_REGION : Région AWS, récupérée depuis les secrets du repository.
Job deploy:

Exécute sur la dernière version d'Ubuntu.
Utilise Bash comme shell et exécute les commandes dans le répertoire racine.
Étapes du Workflow:

Git Checkout: Récupère le code du repository.
Configuration des Crédentials AWS: Configure les credentials AWS en assumant un rôle IAM spécifique.
Setup Terraform: Installe Terraform (version spécifiée).
Terraform fmt: Vérifie le formatage du code Terraform.
Terraform Init: Initialise Terraform avec les configurations du backend spécifiées (bucket S3 et clé).
Terraform Validate: Valide la configuration Terraform.
Terraform Plan: Génère un plan d'exécution Terraform. Cette étape s'exécute uniquement pour les Pull Requests.
Script GitHub: Publie un commentaire sur la Pull Request avec le résultat des étapes précédentes (format, init, validate, plan). Utilise l'action actions/github-script.
Terraform Plan Status: Vérifie le statut du plan Terraform. Si plan échoue, le workflow se termine avec une erreur.
Terraform Apply: Applique les changements Terraform. Cette étape s'exécute uniquement pour les push sur la branche main.



Composants Créés
Rôle IAM: Un rôle est créé pour gérer l'accès et les permissions.
OIDC Provider: Pour une authentification sécurisée avec AWS.
Bucket S3: Utilisé pour le stockage des objets et des états Terraform.
Configurations SSM: Pour gérer les paramètres et secrets.
Politiques IAM
Voici les politiques IAM utilisées pour chaque composant :

Politique pour le Rôle IAM
json
Copy code
{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::YOUR_ACCOUNT_NUMBER:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_USERNAME/YOUR_REPO_NAME:"
                }
            }
        }
    ]
}
Politique pour le Bucket S3
json
Copy code
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::YOUR_BUCKET/",
                "arn:aws:s3:::YOUR_BUCKET"
            ]
        }
    ]
}
Politique pour SSM
json
Copy code
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "ssm:PutParameter",
                "ssm:LabelParameterVersion",
                "ssm:DeleteParameter",
                "ssm:UnlabelParameterVersion",
                "ssm:DescribeParameters",
                "ssm:GetParameterHistory",
                "ssm:ListTagsForResource",
                "ssm:GetParametersByPath",
                "ssm:GetParameters",
                "ssm:GetParameter",
                "ssm:DeleteParameters"
            ],
            "Resource": "*"
        }
    ]
}
Secrets GitHub
Les secrets suivants doivent être déclarés dans GitHub pour le workflow :

AWS_REGION : Région AWS (par exemple, eu-west-3).
AWS_ROLE : Le nom du rôle créé sur AWS.
AWS_BUCKET_NAME : Le nom du bucket S3.
AWS_BUCKET_KEY_NAME : Clé pour l'état Terraform (infra.tfstate).
