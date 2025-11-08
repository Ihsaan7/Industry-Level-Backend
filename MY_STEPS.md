# MY STEPS 

### 1. CREATE THE FILE STRUCTURE  
##### => folders and main.js app.js constant.js and in the root folder .evn file 

### 2. SETUP THE PACKAGE JSON AND WRITE BASIC FOR MAINJS
#### => make changes in the pakcgesJson file like tpye module , dev = nodemon /src/mianjs start node src/main.js
#### => in MainJs import and conifg the DOTENV 

### 3. WRITE TESTING SERVER ROUTE IN APP.js
#### => in the app.js import app.use Middleware and create a simple /health route to check the server
#### => make sure the app.listen on main.js ( and export the app.js ) 
#### => can write the 404 and Centralized Error Handling routes

### 4. SETTING UP THE DB FOR MONGOOSE | STARTING DB IN MAIN.js
#### => Make connection with mongoose URI inside try catch block and import that fucntion
#### => in Main.js call the fucntion wiht then and catch ( and listen the Port )

### 5. UTILS | ASYNCHANDLER | APIERROR | APIRESPONSE
#### => write aysnhanlder for aysnc errors simple higher order function with Promise.resolve and catch(next)
#### => ApiError class extends from Error with few cosntructors (message , statusCode , errors=[] , stack="" | super of message and this. = override
#### => not to forget if(stack) then this.stack = stack else Error.captureStackTrace(this. this.cosntrcutor )
#### => ApiResponse simpler class with same cosntructor and no super where data = data | scuess : statusCode < 400
#### => test the Asynchandler and ApiError by creating Testing Error rotue in App.js
#### => can --> 6.5

### 6. MODELS | USER MODEL
#### => First write user model and inisde it make sure to have pre.save( to save password using bcrtp ) | also comapre method.isPassCorrect
#### => Create method for Gen AccessToken and Refrehsn Token with jwtSign ( refrehs token only sends _id ) | Export model
#### => Can write other models too ( main is User's )
### 6.5 FILE UPLOAD
#### => If there is file Uploading then first configure that in UTILs folder ( cloudinary ) usiing try catch | with cloduinary config(env)
#### =>  simpley get the response and send the response  and unlink fs

### 7. CONTROLLERS | USER.CONTROLLER
#### => go Stepwise = req.body ( valodate -> existed user -> uplaoding file -> create user -> fetch user (without pass)
#### =>  ->
