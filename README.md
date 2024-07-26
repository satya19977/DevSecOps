# DevSecOps
## This is an end-to-end implementation of security practices across the Application(OWASP Juice Shop), Infrastructure (AWS) and Platform (Kubernetes)

<details>
<summary><h2>1 - Application Security</h2></summary>

<details>
<summary><h3>1.1 - Application Vulnerability Scanning </h3></summary>

<h3>Infra Diagram</h3>

    [Include an infrastructure diagram specific to application security]



<h3>Objective</h3>

    Integrate practices into our pipeline to check if our code exposes passwords, tokens, scans application for vulnerabilities such as SQL Injections, XSS Scripting etc

<h3>Code</h3>

         
       stages:
        - cache
        - test
        - build

    ## We use caching to speed up the build process 
    create_cache:
        image: node:18-bullseye
        stage: cache
        script:
            - yarn install
        cache:
            policy: pull-push
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
    
    yarn_test:
        image: node:18-bullseye
        stage: test
        
        script:
            - yarn install
            - yarn test
        cache:
            policy: pull
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
                
    
    ## This stage scans the code for sensitive information such as passwords, token
    gitleaks:
        stage: test
        image:
             name: zricethezav/gitleaks   ## Using image: zricethezav/gitleaks alone is simpler and works well if the default entrypoint of the image is suitable for your needs. 
                                            If you encounter any conflicts or need more control, specifying the entrypoint ensures that your script commands executes as intended.
             entrypoint: [""]
             
             
        ## This srcipt generates a file called gitleaks.json for visualizing vulnerabilities
        script:
            - gitleaks detect --verbose . -f json -r gitleaks.json
            
        ## This is set to true because we don't want the job to end.
        allow_failure: true
        

    ##This stage allows us to scan the code itsel for vulberabilities using SAST tools such as njsscan for cross-site scripting, SQL injection, phising attacks, DDos attack
    njsscan:
    stage: test
    image: python
    before_script:
        - pip3 install --upgrade njsscan
        ## The --exit-warning fails the build 
    script:
        - njsscan --exit-warning . --sarif -o njsscan.sarif
    allow_failure: true
    artifacts:
        when: always
        paths:
            - njsscan.sarif
        



    ##Why another SAST tool? It is because we need to use multiple tools for wider code coverage and certain tools can unearth certain vulnerabilities better than  the other
    semgrep:
        stage: test
        image: returntocorp/semgrep
        ## This basically tell to scan java code
        variables:
            SEMGREP_RULES: p/javascript
        script:
            - semgrep ci --json --output semgrep.json
        allow_failure: true
        artifacts:
            when: always
            paths:
                - semgrep.json

    
      


<h3>Findings from Gitleaks</h3>

      Finding:     password: 'bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI='
      Secret:      bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI=
      RuleID:      generic-api-key
      Entropy:     4.329240
      File:        data/static/users.yml
      Line:        88
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:88
      Finding:     totpSecret: IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
        key: timo
      Secret:      IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
      RuleID:      generic-api-key
      Entropy:     4.351410
      File:        data/static/users.yml
      Line:        150
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:150
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...
      RuleID:      jwt
      Entropy:     5.444070
      File:        frontend/src/app/app.guard.spec.ts
      Line:        40
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:frontend/src/app/app.guard.spec.ts:jwt:40
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...
      RuleID:      jwt
      Entropy:     5.494293
      File:        frontend/src/app/last-login-ip/last-login-ip.component.spec.ts
      Line:        50
      
### Finding 1: Generic API Key Exposure

This finding indicates that an API key is hardcoded in the data/static/users.yml file. The high entropy value suggests that the string is not random text, but likely sensitive information such as a password or an API key. Hardcoding secrets in the source code is a major security risk as it can easily be extracted by anyone with access to the codebase

<h3>Remediation</h3>

Remove the hardcoded API key from the source code.
Use environment variables or secret management tools like AWS Secrets Manager or HashiCorp Vault to manage and access sensitive information securely.

### Finding 2: JWT Token Exposure

A JSON Web Token (JWT) is exposed in the frontend/src/app/app.guard.spec.ts file. JWT tokens are used for authentication and can contain sensitive information. Exposure of JWT tokens can allow attackers to impersonate users or gain unauthorized access to the system.

<h3>Remediation</h3>

Remove the hardcoded JWT token from the source code.
Implement secure storage practices for tokens and ensure they are transmitted securely over the network (e.g., using HTTPS).

### Conclusion

The findings from the Gitleaks scan highlight critical security vulnerabilities related to hardcoded secrets and tokens in the application code. To enhance the security posture of the application, it is essential to remove these hardcoded secrets and implement secure storage and management practices.

</details>    




<details>
<summary><h3>1.2 - Vulnerability Management and Remediation  </h3></summary>
  
<h3>Infra Diagram</h3>

    [Include an infrastructure diagram specific to application security]



<h3>Objective</h3>

    Generate detailed reports highlighting vulnerabilities or security issues found and 
    resolve the security vulnerabilities found during the DevSecOps process to improve the application's security posture.

<h3>Code</h3>

         
       stages:
      - cache
      - test
      - build
      
    
    create_cache:
      image: node:18-bullseye
      stage: cache
      script:
        - yarn install
      cache:
        key:
          files:
            - yarn.lock
        paths:
          - node_modules/
          - yarn.lock
          - .yarn
        policy: pull-push
        
    
    yarn_test:
      image: node:18-bullseye
      stage: test
      script:
        - yarn install
        - yarn test
      cache:
        key:
          files:
            - yarn.lock
        paths:
          - node_modules/
          - yarn.lock
          - .yarn
        policy: pull

    
    gitleaks:
      stage: test
      image:
        name: zricethezav/gitleaks
        entrypoint: [""]
      script:
        - gitleaks detect --verbose --source . -f json -r gitleaks.json
      allow_failure: true
      artifacts:
        when: always
        paths:
          - gitleaks.json
          
    
    njsscan:
      stage: test
      image: python
      before_script:
        - pip3 install --upgrade njsscan
      script:
        - njsscan --exit-warning . --sarif -o njsscan.sarif
      allow_failure: true
      artifacts:
        when: always
        paths:
          - njsscan.sarif
          
    
    semgrep:
      stage: test
      image: returntocorp/semgrep
      variables:
        SEMGREP_RULES: p/javascript
      script:
        - semgrep ci --json --output semgrep.json
      allow_failure: true
      artifacts:
        when: always
        paths:
          - semgrep.json

          
    ## We use Defectdojo to upload our findings from Gitleaks, Semgrep and Njsscan
    upload_reports:
      stage: test
      image: python
      needs: ["gitleaks", "njsscan", "semgrep"]
      when: always
      before_script:
        - pip3 install requests
      script:
        - python3 upload-reports.py gitleaks.json
        - python3 upload-reports.py njsscan.sarif
        - python3 upload-reports.py semgrep.json

        
    
    build_image:
      stage: build
      image: docker:24
      services:
        - docker:24-dind
      variables:
        DOCKER_PASS: $DOCKER_PASS
        DOCKER_USER: $DOCKER_USER
      before_script:
        - echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
      script:
        - docker build -t $IMAGE_NAME:$IMAGE_TAG .
        - docker push $IMAGE_NAME:$IMAGE_TAG

    
<h3>Python code to automate report feeding to Defectdojo</h3>
  import requests
    import sys
    
    file_name = sys.argv[1]
    scan_type = ''
    
    if file_name == 'gitleaks.json':
        scan_type = 'Gitleaks Scan'
    elif file_name == 'njsscan.sarif':
        scan_type = 'SARIF'
    elif file_name == 'semgrep.json':
        scan_type = 'Semgrep JSON Report'
    
    
    headers = {
        'Authorization': 'Token e71f520d6cb842d4465dab1b1d9b97e04d7a231f'
    }
    
    url = 'https://demo.defectdojo.org/api/v2/import-scan/'
    
    data = {
        'active': True,
        'verified': True,
        'scan_type': scan_type,
        'minimum_severity': 'Low',
        'engagement': 19
    }
    
    files = {
        'file': open(file_name, 'rb')
    }
    
    response = requests.post(url, headers=headers, data=data, files=files)
    
    if response.status_code == 201:
        print('Scan results imported successfully')
    else:
        print(f'Failed to import scan results: {response.content}')





<h3>Findings</h3>

      
![Screenshot (121)](https://github.com/user-attachments/assets/ab57f95a-cb80-4f53-a4d5-cfc16dd3eadc)


![Screenshot (122)](https://github.com/user-attachments/assets/2cc3cc54-fbed-441f-be10-55977ff4417d)



### Responsibilities 
After these vulnerabilities are detected, it's the job of the developer (since it's the code written by them) to remediate the issue.    
      
### Finding 1:  SQL Injection

This finding by Semgrep report suggests that this is a High severity issue leading to SQL Injection. This vulnerability occurs when an attacker can manipulate the input fields to inject malicious SQL code, which can be executed by the database.

Example: A hacker can steal or manipulate data inside a database.


### Remediation

#### Before code fix

Directly inserting user input into SQL statements without using parameterized queries leaves the code open to injection vulnerabilities.

```
const injectionChars = /"|'|;|and|or|;|#/i;

module.exports = function searchProducts () {
  return (req: Request, res: Response, next: NextFunction) => {
    let criteria: any = req.query.q === 'undefined' ? '' : req.query.q ?? ''
    criteria = (criteria.length <= 200) ? criteria : criteria.substring(0, 200)
    if (criteria.match(injectionChars)) {
      res.status(400).send()
      return
    }
    models.sequelize.query("SELECT * FROM Products WHERE ((name LIKE '%"+criteria+"%' OR description LIKE '%"+criteria+"%') AND deletedAt IS NULL) ORDER BY name")
      .then(([products]: any) => {
        const dataString = JSON.stringify(products)
        for (let i = 0; i < products.length; i++) {
          products[i].name = req.__(products[i].name)
          products[i].description = req.__(products[i].description)
        }
        res.json(utils.queryResultToJson(products))
      }).catch((error: ErrorWithParent) => {
        next(error.parent)
      })
  }
}

```

#### After code fix

The updated query now uses a named parameter ":searchQuery". By using parameterized queries, the code effectively protects against SQL injection

```
const injectionChars = /"|'|;|and|or|;|#/i;

module.exports = function searchProducts () {
  return (req: Request, res: Response, next: NextFunction) => {
    let criteria: any = req.query.q === 'undefined' ? '' : req.query.q ?? ''
    criteria = (criteria.length <= 200) ? criteria : criteria.substring(0, 200)
    if (criteria.match(injectionChars)) {
      res.status(400).send()
      return
    }
    models.sequelize.query("SELECT * FROM Products WHERE ((name LIKE :searchQuery OR description LIKE :searchQuery) AND deletedAt IS NULL) ORDER BY name", {
      replacements: { searchQuery: '%' + criteria + '%' },
      type: models.Sequelize.QueryTypes.SELECT
    })
      .then(([products]: any) => {
        const dataString = JSON.stringify(products)
        for (let i = 0; i < products.length; i++) {
          products[i].name = req.__(products[i].name)
          products[i].description = req.__(products[i].description)
        }
        res.json(utils.queryResultToJson(products))
      }).catch((error: ErrorWithParent) => {
        next(error.parent)
      })
  }
}

```



### Finding 2 : Weak Hash

![Screenshot (119)](https://github.com/user-attachments/assets/3a238e7c-4c32-4454-8b95-9737eabfad1a)

The algorithm has known flaws that can be exploited by attackers to break the encryption and gain unauthorized access to sensitive data.


### Remediation

#### Before code fix

The code uses MD5 algorithm which is vulnerable to collision attacks, where two different inputs produce the same hash output. This weakness can be exploited to forge digital signatures and certificates.

```
grunt.registerTask('checksum', 'Create .md5 checksum files', function () {
    const fs = require('fs')
    const crypto = require('crypto')
    fs.readdirSync('dist/').forEach(file => {
      const buffer = fs.readFileSync('dist/' + file)
      const md5 = crypto.createHash('md5')
      md5.update(buffer)
      const md5Hash = md5.digest('hex')
      const md5FileName = 'dist/' + file + '.md5'
      grunt.file.write(md5FileName, md5Hash)
      grunt.log.write(`Checksum ${md5Hash} written to file ${md5FileName}.`).verbose.write('...').ok()
      grunt.log.writeln()
    })
  })

```
#### After code fix

We have replaced MD5 with a stronger algorithm SHA256

```
grunt.registerTask('checksum', 'Create .sha256 checksum files', function () {
    const fs = require('fs')
    const crypto = require('crypto')
    fs.readdirSync('dist/').forEach(file => {
      const buffer = fs.readFileSync('dist/' + file)
      const sha256 = crypto.createHash('sha256')
      sha256.update(buffer)
      const sha256Hash = sha256.digest('hex')
      const sha256FileName = 'dist/' + file + '.sha256'
      grunt.file.write(sha256FileName, sha256Hash)
      grunt.log.write(`Checksum ${sha256Hash} written to file ${sha256FileName}.`).verbose.write('...').ok()
      grunt.log.writeln()
    })
  })

```




</details>


<details>
<summary><h3>1.3 - Vulnerability Scanning for Application Dependencies   </h3></summary>
  
<h3>Infra Diagram</h3>

    [Include an infrastructure diagram specific to application security]



<h3>Objective</h3>

    Integrate GitLeaks into our pipeline to check if our code exposes passwords, tokens and any other credentials

<h3>Code</h3>

         
       stages:
        - cache
        - test
        - build

    ## We use caching to speed up the build process 
    create_cache:
        image: node:18-bullseye
        stage: cache
        script:
            - yarn install
        cache:
            policy: pull-push
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
    
    yarn_test:
        image: node:18-bullseye
        stage: test
        
        script:
            - yarn install
            - yarn test
        cache:
            policy: pull
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
                
    
    ## This stage scans the code for sensitive information such as passwords, token
    gitleaks:
        stage: test
        image:
             name: zricethezav/gitleaks   ## Using image: zricethezav/gitleaks alone is simpler and works well if the default entrypoint of the image is suitable for your needs. 
                                            If you encounter any conflicts or need more control, specifying the entrypoint ensures that your script commands executes as intended.
             entrypoint: [""]
             
             
        ## This srcipt generates a file called gitleaks.json for visualizing vulnerabilities
        script:
            - gitleaks detect --verbose . -f json -r gitleaks.json
            
        ## This is set to true because we don't want the job to end.
        allow_failure: true
    
      




<h3>Findings</h3>

      Finding:     password: 'bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI='
      Secret:      bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI=
      RuleID:      generic-api-key
      Entropy:     4.329240
      File:        data/static/users.yml
      Line:        88
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:88
      Finding:     totpSecret: IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
        key: timo
      Secret:      IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
      RuleID:      generic-api-key
      Entropy:     4.351410
      File:        data/static/users.yml
      Line:        150
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:150
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...
      RuleID:      jwt
      Entropy:     5.444070
      File:        frontend/src/app/app.guard.spec.ts
      Line:        40
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:frontend/src/app/app.guard.spec.ts:jwt:40
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...
      RuleID:      jwt
      Entropy:     5.494293
      File:        frontend/src/app/last-login-ip/last-login-ip.component.spec.ts
      Line:        50
      
### Finding 1: Generic API Key Exposure

This finding indicates that an API key is hardcoded in the data/static/users.yml file. The high entropy value suggests that the string is not random text, but likely sensitive information such as a password or an API key. Hardcoding secrets in the source code is a major security risk as it can easily be extracted by anyone with access to the codebase

<h3>Remediation</h3>

Remove the hardcoded API key from the source code.
Use environment variables or secret management tools like AWS Secrets Manager or HashiCorp Vault to manage and access sensitive information securely.

### Finding 2: JWT Token Exposure

A JSON Web Token (JWT) is exposed in the frontend/src/app/app.guard.spec.ts file. JWT tokens are used for authentication and can contain sensitive information. Exposure of JWT tokens can allow attackers to impersonate users or gain unauthorized access to the system.

<h3>Remediation</h3>

Remove the hardcoded JWT token from the source code.
Implement secure storage practices for tokens and ensure they are transmitted securely over the network (e.g., using HTTPS).

### Conclusion

The findings from the Gitleaks scan highlight critical security vulnerabilities related to hardcoded secrets and tokens in the application code. To enhance the security posture of the application, it is essential to remove these hardcoded secrets and implement secure storage and management practices.

</details>


<details>
<summary><h3>1.4 - Build a CD Pipeline   </h3></summary>
  
<h3>Infra Diagram</h3>

    [Include an infrastructure diagram specific to application security]



<h3>Objective</h3>

    Integrate GitLeaks into our pipeline to check if our code exposes passwords, tokens and any other credentials

<h3>Code</h3>

         
       stages:
        - cache
        - test
        - build

    ## We use caching to speed up the build process 
    create_cache:
        image: node:18-bullseye
        stage: cache
        script:
            - yarn install
        cache:
            policy: pull-push
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
    
    yarn_test:
        image: node:18-bullseye
        stage: test
        
        script:
            - yarn install
            - yarn test
        cache:
            policy: pull
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
                
    
    ## This stage scans the code for sensitive information such as passwords, token
    gitleaks:
        stage: test
        image:
             name: zricethezav/gitleaks   ## Using image: zricethezav/gitleaks alone is simpler and works well if the default entrypoint of the image is suitable for your needs. 
                                            If you encounter any conflicts or need more control, specifying the entrypoint ensures that your script commands executes as intended.
             entrypoint: [""]
             
             
        ## This srcipt generates a file called gitleaks.json for visualizing vulnerabilities
        script:
            - gitleaks detect --verbose . -f json -r gitleaks.json
            
        ## This is set to true because we don't want the job to end.
        allow_failure: true
    
      




<h3>Findings</h3>

      Finding:     password: 'bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI='
      Secret:      bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI=
      RuleID:      generic-api-key
      Entropy:     4.329240
      File:        data/static/users.yml
      Line:        88
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:88
      Finding:     totpSecret: IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
        key: timo
      Secret:      IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
      RuleID:      generic-api-key
      Entropy:     4.351410
      File:        data/static/users.yml
      Line:        150
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:150
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...
      RuleID:      jwt
      Entropy:     5.444070
      File:        frontend/src/app/app.guard.spec.ts
      Line:        40
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:frontend/src/app/app.guard.spec.ts:jwt:40
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...
      RuleID:      jwt
      Entropy:     5.494293
      File:        frontend/src/app/last-login-ip/last-login-ip.component.spec.ts
      Line:        50
      
### Finding 1: Generic API Key Exposure

This finding indicates that an API key is hardcoded in the data/static/users.yml file. The high entropy value suggests that the string is not random text, but likely sensitive information such as a password or an API key. Hardcoding secrets in the source code is a major security risk as it can easily be extracted by anyone with access to the codebase

<h3>Remediation</h3>

Remove the hardcoded API key from the source code.
Use environment variables or secret management tools like AWS Secrets Manager or HashiCorp Vault to manage and access sensitive information securely.

### Finding 2: JWT Token Exposure

A JSON Web Token (JWT) is exposed in the frontend/src/app/app.guard.spec.ts file. JWT tokens are used for authentication and can contain sensitive information. Exposure of JWT tokens can allow attackers to impersonate users or gain unauthorized access to the system.

<h3>Remediation</h3>

Remove the hardcoded JWT token from the source code.
Implement secure storage practices for tokens and ensure they are transmitted securely over the network (e.g., using HTTPS).

### Conclusion

The findings from the Gitleaks scan highlight critical security vulnerabilities related to hardcoded secrets and tokens in the application code. To enhance the security posture of the application, it is essential to remove these hardcoded secrets and implement secure storage and management practices.

</details>

<details>
<summary><h3>1.5 - Image Scanning - Build Secure Docker Images    </h3></summary>
  
<h3>Infra Diagram</h3>

    [Include an infrastructure diagram specific to application security]



<h3>Objective</h3>

    Integrate GitLeaks into our pipeline to check if our code exposes passwords, tokens and any other credentials

<h3>Code</h3>

         
       stages:
        - cache
        - test
        - build

    ## We use caching to speed up the build process 
    create_cache:
        image: node:18-bullseye
        stage: cache
        script:
            - yarn install
        cache:
            policy: pull-push
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
    
    yarn_test:
        image: node:18-bullseye
        stage: test
        
        script:
            - yarn install
            - yarn test
        cache:
            policy: pull
            key:
                files:
                    - yarn.lock
            paths:
                - node_modules/
                - yarn.lock
                - .yarn
                
    
    ## This stage scans the code for sensitive information such as passwords, token
    gitleaks:
        stage: test
        image:
             name: zricethezav/gitleaks   ## Using image: zricethezav/gitleaks alone is simpler and works well if the default entrypoint of the image is suitable for your needs. 
                                            If you encounter any conflicts or need more control, specifying the entrypoint ensures that your script commands executes as intended.
             entrypoint: [""]
             
             
        ## This srcipt generates a file called gitleaks.json for visualizing vulnerabilities
        script:
            - gitleaks detect --verbose . -f json -r gitleaks.json
            
        ## This is set to true because we don't want the job to end.
        allow_failure: true
    
      




<h3>Findings</h3>

      Finding:     password: 'bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI='
      Secret:      bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI=
      RuleID:      generic-api-key
      Entropy:     4.329240
      File:        data/static/users.yml
      Line:        88
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:88
      Finding:     totpSecret: IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
        key: timo
      Secret:      IFTXE3SPOEYVURT2MRYGI52TKJ4HC3KH
      RuleID:      generic-api-key
      Entropy:     4.351410
      File:        data/static/users.yml
      Line:        150
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:data/static/users.yml:generic-api-key:150
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE...
      RuleID:      jwt
      Entropy:     5.444070
      File:        frontend/src/app/app.guard.spec.ts
      Line:        40
      
      Fingerprint: c3340cda147c54325dbf3b32fc863f3402caa5da:frontend/src/app/app.guard.spec.ts:jwt:40
      Finding:     ...e.setItem('token', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...'
      Secret:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7Imxhc3RMb2dpbklwIjoiMS4yLjMuNCJ9fQ.RAkmdqwNypuOxv3S...
      RuleID:      jwt
      Entropy:     5.494293
      File:        frontend/src/app/last-login-ip/last-login-ip.component.spec.ts
      Line:        50
      
### Finding 1: Generic API Key Exposure

This finding indicates that an API key is hardcoded in the data/static/users.yml file. The high entropy value suggests that the string is not random text, but likely sensitive information such as a password or an API key. Hardcoding secrets in the source code is a major security risk as it can easily be extracted by anyone with access to the codebase

<h3>Remediation</h3>

Remove the hardcoded API key from the source code.
Use environment variables or secret management tools like AWS Secrets Manager or HashiCorp Vault to manage and access sensitive information securely.

### Finding 2: JWT Token Exposure

A JSON Web Token (JWT) is exposed in the frontend/src/app/app.guard.spec.ts file. JWT tokens are used for authentication and can contain sensitive information. Exposure of JWT tokens can allow attackers to impersonate users or gain unauthorized access to the system.

<h3>Remediation</h3>

Remove the hardcoded JWT token from the source code.
Implement secure storage practices for tokens and ensure they are transmitted securely over the network (e.g., using HTTPS).

### Conclusion

The findings from the Gitleaks scan highlight critical security vulnerabilities related to hardcoded secrets and tokens in the application code. To enhance the security posture of the application, it is essential to remove these hardcoded secrets and implement secure storage and management practices.

</details>

</details>



<details>
<summary><h2>2 - Infrastructure Security</h2></summary>

(Add Infrastructure Security content here)
</details>
<details>
    
<summary><h2>3 - Platform Security</h2></summary>

(Add Platform Security content here)
</details>





