pipeline {
    agent any

    parameters {
        choice(name: 'AUTH_TYPE', choices: ['token', 'ssh'], description: 'Authentication type for cloning and pushing')
        string(name: 'SOURCE_PLATFORM', defaultValue: '', description: 'Source platform (bitbucket/github)')
        string(name: 'DEST_PLATFORM', defaultValue: '', description: 'Destination platform (bitbucket/github)')
        string(name: 'SOURCE_USERNAME', defaultValue: '', description: 'Source username')
        string(name: 'DEST_USERNAME', defaultValue: '', description: 'Destination username')
        string(name: 'SOURCE_WORKSPACE', defaultValue: '', description: 'Source workspace or organization')
        string(name: 'DEST_ORGANIZATION', defaultValue: '', description: 'Destination organization or user account')
        booleanParam(name: 'USE_SSH', defaultValue: false, description: 'Use SSH for cloning and pushing?')
        password(name: 'SOURCE_TOKEN', defaultValue: '', description: 'Token for source platform')
        password(name: 'DEST_TOKEN', defaultValue: '', description: 'Token for destination platform')
        text(name: 'SOURCE_SSH_KEY', defaultValue: '', description: 'SSH key for source platform (base64 encoded)')
        text(name: 'DEST_SSH_KEY', defaultValue: '', description: 'SSH key for destination platform (base64 encoded)')
    }

    stages {
        stage('Sync Repositories') {
            steps {
                script {
                    def authType = params.AUTH_TYPE
                    def sourcePlatform = params.SOURCE_PLATFORM
                    def destPlatform = params.DEST_PLATFORM
                    def sourceUsername = params.SOURCE_USERNAME
                    def sourceToken = params.SOURCE_TOKEN
                    def destUsername = params.DEST_USERNAME
                    def destToken = params.DEST_TOKEN
                    def sourceWorkspace = params.SOURCE_WORKSPACE
                    def destOrganization = params.DEST_ORGANIZATION
                    def useSsh = params.USE_SSH
                    def sourceSshKey = params.SOURCE_SSH_KEY
                    def destSshKey = params.DEST_SSH_KEY

                    def createRepoUrl = { platform, workspace, repo, useSsh, username, token ->
                        if (useSsh) {
                            return platform == 'bitbucket' ? "git@bitbucket.org:${workspace}/${repo}.git" : "git@github.com:${workspace}/${repo}.git"
                        } else {
                            return platform == 'bitbucket' ? "https://${username}:${token}@bitbucket.org/${workspace}/${repo}.git" : "https://${username}:${token}@github.com/${workspace}/${repo}.git"
                        }
                    }

                    if (authType == 'ssh') {
                        if (sourceSshKey) {
                            writeFile(file: '/root/.ssh/id_rsa', text: sourceSshKey, encoding: 'base64')
                            sh 'chmod 600 /root/.ssh/id_rsa'
                        }
                        if (destSshKey) {
                            writeFile(file: '/root/.ssh/id_rsa_dest', text: destSshKey, encoding: 'base64')
                            sh 'chmod 600 /root/.ssh/id_rsa_dest'
                            sh 'ssh-keyscan github.com >> /root/.ssh/known_hosts'
                            sh 'ssh-keyscan bitbucket.org >> /root/.ssh/known_hosts'
                        }
                    }

                    echo "DEBUG: Source platform is '${sourcePlatform}'"
                    echo "DEBUG: Destination platform is '${destPlatform}'"

                    // Fetch list of repositories
                    def repos
                    if (sourcePlatform == 'bitbucket') {
                        repos = sh(script: "curl -s -u '${sourceUsername}:${sourceToken}' 'https://api.bitbucket.org/2.0/repositories/${sourceWorkspace}' | grep -oP '(?<=\"name\": \")[^\"]*'", returnStdout: true).trim()
                    } else {
                        repos = sh(script: "curl -s -u '${sourceUsername}:${sourceToken}' 'https://api.github.com/users/${sourceWorkspace}/repos' | grep -oP '(?<=\"name\": \")[^\"]*'", returnStdout: true).trim()
                    }

                    if (repos.isEmpty()) {
                        error "No repositories found or authentication failed."
                    }

                    // Sync each repository
                    repos.split('\n').each { repo ->
                        repo = repo.trim()
                        echo "Syncing repository '${repo}'..."

                        // Clean up the directory if it already exists
                        sh "rm -rf '${repo}.git' || true"

                        def sourceRepoUrl = createRepoUrl(sourcePlatform, sourceWorkspace, repo, useSsh, sourceUsername, sourceToken)
                        sh "git clone --bare '${sourceRepoUrl}' '${repo}.git'"

                        dir("${repo}.git") {
                            def destRepoUrl = createRepoUrl(destPlatform, destOrganization, repo, useSsh, destUsername, destToken)

                            def repoExists
                            if (destPlatform == 'bitbucket') {
                                repoExists = sh(script: "curl -s -u '${destUsername}:${destToken}' 'https://api.bitbucket.org/2.0/repositories/${destOrganization}/${repo}' || true", returnStdout: true).trim()
                                if (!repoExists) {
                                    echo "Creating repository '${repo}' on Bitbucket..."
                                    sh "curl -s -u '${destUsername}:${destToken}' 'https://api.bitbucket.org/2.0/repositories/${destOrganization}/${repo}' -X POST -H 'Content-Type: application/json' -d '{\"scm\": \"git\", \"is_private\": true}'"
                                }
                            } else {
                                repoExists = sh(script: "curl -s -u '${destUsername}:${destToken}' 'https://api.github.com/repos/${destOrganization}/${repo}' || true", returnStdout: true).trim()
                                if (!repoExists) {
                                    echo "Creating repository '${repo}' on GitHub..."
                                    sh "curl -s -u '${destUsername}:${destToken}' 'https://api.github.com/user/repos' -d '{\"name\":\"${repo}\"}'"
                                }
                            }

                            echo "Pushing changes to '${repo}' on ${destPlatform}..."
                            sh "git remote add source '${sourceRepoUrl}'"
                            sh "git remote add destination '${destRepoUrl}'"
                            sh "git push --all destination"
                            sh "git push --tags destination"
                        }
                        sh "rm -rf '${repo}.git'"
                    }

                    echo "All repositories have been synced."
                }
            }
        }
    }
}
