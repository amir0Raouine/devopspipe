stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('sonar') {
            sh 'mvn -DskipTests clean verify sonar:sonar'
        }
    }
}
