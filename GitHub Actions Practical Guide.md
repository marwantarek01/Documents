# Strategy matrix overview
GitHub Actions' strategy matrix is a powerful feature that allows you to run jobs in parallel across multiple configurations. it works like a list of variables, where GitHub Actions loops over each combination and runs parallel jobs for every set of variables.
## Usecase
in this usecase scenario Strategy matrix is used in the build process of NodeJS app. specifically to create `package.json` file 
for every microservice.

![micro-svc](https://github.com/marwantarek01/assets/blob/main/micro-svc%20folders.png)
### steps

- define strategy matrix named `microservice` with the all microservices names 
```
 build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        microservice: [accommodationService, customerService, adminService, authenticationService, companyService, notificationService, reservationService]
    
```
- checkout code and Setup Node.js environment
- create package.json by switching to each microservice directory then runnig `npm init -y`

```
  - name: create package.json for ${{ matrix.microservice }}
      working-directory: ./${{ matrix.microservice }}
      run: npm init -y
```
`${{ matrix.microservice }}` refers to variables in microservices strategy matrix .

-------------------------------------
# GitHub artifacts overview
  - `package.json` files created in the previous job are not automatically saved after the job completes. Once the job finishes, the runner is cleaned up, and any generated files are discarded.         
  To persist files across jobs, you need to use the **artifact upload** feature. 
  - You can think of artifacts as a temporary storage on GitHub, primarily used for sharing files between jobs in a workflow. They allow you to pass outputs like build files, logs, or test results from one job to another. This makes workflows more efficient by enabling different jobs to work with the same data without needing to recreate or fetch it again.
## usecase
using artifact upload action to save `package.json` files

### steps

```
 - name: upload package.json to artifact
      uses: actions/upload-artifact@v4
      with:
        name: ${{ matrix.microservice }}                      # name assigned to the uploaded files
        path: ./${{ matrix.microservice }}/package.json       # path to the file you want to upload
```
in this case each `package.json` file is named after the microservice it belongs to

![artifacts](https://github.com/marwantarek01/assets/blob/main/ss%20of%20artifacts.png)

-------------------------------------
# ssh and run commands on server
In this example, the web server is located in an AWS private subnet. To access it, we must first SSH into the jump server, which resides in the public subnet within the same VPC. After connecting to the jump server, SSH access to the web server can be established.
![diagram1](https://github.com/marwantarek01/assets/blob/main/rev-proxy-arh.png)


```
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: ssh to jump-server
        env:
          PRIVATE_KEY: ${{secrets.JUMP_SERVER_PRIVATE_KEY}}
          WEB_SERVER_PRIVATE_KEY: ${{secrets.WEB_SERVER_PRIVATE_KEY}}
          USER_NAME: ${{secrets.USER_NAME}}
          JUMP_SERVER_IP: ${{secrets.JUMP_SERVER_IP}}
        run: |
         echo "$PRIVATE_KEY" > private_key && chmod 600 private_key     
         ssh -o StrictHostKeyChecking=no -i private_key ${USER_NAME}@${JUMP_SERVER_IP} << EOF
                      ssh -T -i server.pem ec2-user@web-server-IP "cd pipeline2 && docker-compose down && docker-compose pull"
         EOF
```
- `echo "$PRIVATE_KEY" > private_key && chmod 600 private_key` prints the value of `PRIVATE_KEY` (jump-server private key) in the file named `private_key` inside githubActions runner. Afterthat, githubActions runner use this file to ssh to the jump-server.
- The `EOF` marker is used to pass multiple commands as input to an SSH command. in this example ssh command is used again to ssh to the web-server. note that any marker can be used not just EOF
  ```
  ssh -o ... << EOF
   # ssh to the web-server
  EOF
  ```
- after that ssh to the web-server using the command `ssh -i server.pem ec2-user@web-server_IP`, **note that* In this step, you cannot use the secrets defined in GitHub Secrets because the process does not run on a GitHub runner. Instead, it runs on the jump server, which has no connection with GitHub Actions secrets.For sensitive information such as usernames and IP addresses, you can set environment variables directly on the jump server and use them.
- double quotes "" is used to encapsulate a series of commands that will be executed on the remote server after the SSH connection is established. 

