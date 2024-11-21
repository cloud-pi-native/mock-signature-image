# Mock signature image

Projet mock concernant la signature d'une image Docker.

## Signature

Le fichier .gitlab-ci.yml décrit une pipeline GitLab qui permet de construire et de pousser une image, d'effectuer une signature et de vérifier cette signature.

![gitlab](/img/pipeline.png)

Chaque stage est composé de deux jobs, l'un pour Apache et l'autre pour Nginx.

Une exception est faite pour le stage read-secret qui permet la récupération des secrets (token Harbor, etc.).

- Docker-build:

    Les deux jobs permettent de construire et de pousser une image Docker. Cependant, celui de Nginx génère un artefact GitLab contenant le hash de l'image.

- Sign:

    ```yaml
    ---
    sign_apache:
      stage: sign
      variables:
        COSIGN_PASSWORD: monsupermdp1234
        REGISTRY_URL: $IMAGE_REPOSITORY
      image: bitnami/cosign:2.4.1-debian-12-r0
      before_script:
        - mkdir -p $HOME/.docker
        - echo "$DOCKER_AUTH" > $HOME/.docker/config.json
        - rm cosign.pub
      script:
        - cosign generate-key-pair
        - cosign sign --key cosign.key $REGISTRY_URL/apache:$CI_COMMIT_SHORT_SHA -y
    ```
    
    Ce job permet de signer l'image Apache. Il utilise une clé privée générée par l'utilitaire Cosign pendant le job.

    Le mot de passe de la clé est fourni par la variable `COSIGN_PASSWORD`.

    Pour signer l'image, nous utilisons son tag, ce qui n'est pas du tout recommandé !
  

    ```yaml
    ---
    sign_nginx:
      stage: sign
      variables:
        COSIGN_PASSWORD: $MON_SUPER_MDP
        REGISTRY_URL: $IMAGE_REPOSITORY
      image: bitnami/cosign:2.4.1-debian-12-r0
      before_script:
        - mkdir -p $HOME/.docker
        - echo "$DOCKER_AUTH" > $HOME/.docker/config.json
      script:
        - IMAGE_DIGEST=$( tail -n 1 digest.txt )
        - cosign sign --key env://MA_SUPER_CLE $IMAGE_DIGEST -y --annotations "Project=$CI_PROJECT_NAME"
    ```

    Comme le précédent, ce job signe l'image Nginx en utilisant le hash de l'image Docker et des variables d'environnement pour la clé et son mot de passe.
    
    Les variables sont renseignées au niveau des paramètres du projet. 
    
    De plus nous avons la possibilité d'ajouter des annotations à notre signature.



    ![vars](/img/variables.png)

- Verify

    ```yaml
    ---
    nginx_check:
      stage: verify
      variables:
        REGISTRY_URL: $IMAGE_REPOSITORY
      image: bitnami/cosign:2.4.1-debian-12-r0
      before_script:
        - mkdir -p $HOME/.docker
        - echo "$DOCKER_AUTH" > $HOME/.docker/config.json
      script:
        - cosign verify --key cosign.pub $REGISTRY_URL/nginx:$CI_COMMIT_SHORT_SHA --annotations "Project=$CI_PROJECT_NAME" --output='text'
    ```

    Pour vérifier une signature, il faut utiliser la clé publique. Il s'agit d'un job à titre indicatif,il n'est pas obligatoire dans votre projet.

    Le job Apache échouera toujours, comme on peut le voir sur l'image ci-dessus, car la clé publique utilisée n'est pas la bonne.
    
    En revanche, cette clé permet de vérifier l'image Nginx. 
    
    Il est également possible de vérifier les annotations de la signature.

    Depuis l'interface Harbor, l'image apparaît avec sa signature en tant qu'artefact.
    
    ![vars](/img/harbor.png)

## Déploiement

La vérification de la signature d'une image doit se faire lors du déploiement. 

Pour cela il faut utiliser une regle Kyverno. Il est possible de renseigner la clé publique dans le manifest ou d'utiliser un secret contenat la clé.

Le dossier infra contient des manifests:

- `apache-wrong.yaml` --> Déploie l'image Apache signée.

- `nginx-hash.yaml` -->  Déploie l'image Nginx signée en utilisant le hash.

- `nginx-io.yaml` --> Déploie une image Nginx publique.

- `nginx-tag.yaml` --> Déploie l'image Nginx signée en utilisant le tag.

- `kyverno-verify` --> Déploie une policy Kyverno qui vérifie la signature des images provenant de Harbor avec la clé publique présente sur le repository.

- `secret.sops.enc.yaml` --> Déploie un secret contenant la clé publique

Voici le résultat des déploiements depuis Argo-CD:

![vars](/img/argo.png)

Comme pour la CI, nous ne pourrons pas déployer l'image Apache car la clé publique n'est pas correcte.