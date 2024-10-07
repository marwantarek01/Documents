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
