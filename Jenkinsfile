// Build pipeline voor de todo app (labo 1 infra)
// Jenkins draait zelf in een container en gebruikt de Docker daemon van infra010,
// dus alle containers hieronder draaien naast Jenkins op de host.
pipeline {
    agent any

    triggers {
        // GitHub kan infra010 (172.16.0.10, host-only netwerk) niet bereiken voor een webhook,
        // dus Jenkins kijkt zelf om de 2 minuten of er nieuwe commits op main zijn
        pollSCM('H/2 * * * *')
    }

    environment {
        APP_CONTAINER = 'todoapp'
        APP_PORT      = '8081'   // 8080 is al in gebruik door Jenkins
        DB_CONTAINER  = 'todoappdb'
        NETWORK       = 'todoapp-net'
        TEST_DB       = "todoapp-testdb-${env.BUILD_NUMBER}"
        DB_USER       = 'todo_usr'
        DB_PASSWORD   = 'letmeinplz'
    }

    stages {
        stage('Unit tests') {
            steps {
                // aparte wegwerp-database voor de tests
                sh '''
                    docker run -d --name $TEST_DB \
                      -e MARIADB_ROOT_PASSWORD=sekrit \
                      -e MARIADB_DATABASE=todo_test_db \
                      -e MARIADB_USER=$DB_USER -e MARIADB_PASSWORD=$DB_PASSWORD \
                      mariadb:11
                    for i in $(seq 1 30); do
                      docker exec $TEST_DB healthcheck.sh --connect --innodb_initialized && break
                      sleep 2
                    done
                    docker exec -i $TEST_DB mariadb -u$DB_USER -p$DB_PASSWORD todo_test_db < TodoApp/schema.sql
                '''
                // --volumes-from: de workspace staat in het volume van jenkins_server, zo ziet de
                // SDK-container dezelfde bestanden op hetzelfde pad.
                // --network container:...: localhost:3306 uit appsettings.json is dan de test-db
                sh '''
                    docker run --rm --volumes-from jenkins_server -w "$WORKSPACE" \
                      --network container:$TEST_DB \
                      -v nuget-cache:/root/.nuget/packages \
                      mcr.microsoft.com/dotnet/sdk:10.0 dotnet test
                '''
            }
            post {
                always {
                    // -v: ook het anonieme volume van de test-db weggooien
                    sh 'docker rm -f -v $TEST_DB || true'
                }
            }
        }

        stage('Build') {
            steps {
                sh 'docker build -t todoapp:$BUILD_NUMBER -t todoapp:latest TodoApp'
            }
        }

        stage('Deploy') {
            steps {
                // database blijft draaien tussen deploys, data staat in volume todoapp-data
                sh '''
                    docker network inspect $NETWORK >/dev/null 2>&1 || docker network create $NETWORK
                    if ! docker container inspect $DB_CONTAINER >/dev/null 2>&1; then
                      docker run -d --name $DB_CONTAINER --network $NETWORK --restart unless-stopped \
                        -e MARIADB_ROOT_PASSWORD=sekrit -e MARIADB_DATABASE=todo_db \
                        -e MARIADB_USER=$DB_USER -e MARIADB_PASSWORD=$DB_PASSWORD \
                        -v todoapp-data:/var/lib/mysql \
                        mariadb:11
                    fi
                    docker start $DB_CONTAINER
                    for i in $(seq 1 30); do
                      docker exec $DB_CONTAINER healthcheck.sh --connect --innodb_initialized && break
                      sleep 2
                    done
                    # schema.sql gebruikt CREATE TABLE IF NOT EXISTS, dus mag elke keer opnieuw
                    docker exec -i $DB_CONTAINER mariadb -u$DB_USER -p$DB_PASSWORD todo_db < TodoApp/schema.sql
                '''
                sh '''
                    docker rm -f $APP_CONTAINER || true
                    docker run -d --name $APP_CONTAINER --network $NETWORK --restart unless-stopped \
                      -p $APP_PORT:8080 \
                      -e "ConnectionStrings__TodoDb=Server=$DB_CONTAINER;Port=3306;Database=todo_db;User=$DB_USER;Password=$DB_PASSWORD;" \
                      todoapp:latest
                '''
            }
        }

        stage('Acceptance test') {
            steps {
                sh '''
                    for i in $(seq 1 15); do
                      curl -sf -o page.html http://172.16.0.10:$APP_PORT/ && break
                      sleep 2
                    done
                    grep "<h1>Todo List</h1>" page.html
                '''
            }
        }
    }
}
