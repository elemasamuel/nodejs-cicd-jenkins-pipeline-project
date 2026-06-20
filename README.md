# DevOps Bootcamp
## Node Project
<br />

This project contains a node.js application used for the exercises in module 08 ("Build Automation & CI/CD with Jenkins") and module 09 ("AWS Services") of Nana Janashia's [DevOps Bootcamp](https://www.techworld-with-nana.com/devops-bootcamp).

### Test
The project uses jest library for tests. (see "test" script in package.json)
There is 1 test (server.test.js) in the project that checks whether the main index.html file exists in the project. 

To run the nodejs test:
```sh
npm run test
```

Make sure to download jest library before running test, otherwise jest command defined in package.json won't be found.
```sh
npm install
```

In order to see failing test, remove index.html or rename it and run tests.