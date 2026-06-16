
                            git config user.email "rajeshchilukuri210@gmail.com"
                            git config user.name "Rajesh-210"
                            BUILD_NUMBER=${BUILD_NUMBER}
                            echo $BUILD_NUMBER
                            imageTag=$(grep -oP '(?<=chatbot:)[^ ]+' deployment.yml)
                            echo $imageTag
                            sed -i "s/chatbot:${imageTag}/chatbot:${BUILD_NUMBER}/" deployment.yml
                            git add deployment.yml
                            git commit -m "Update deployment Image to version ${BUILD_NUMBER}"
                            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                        