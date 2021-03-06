# What?

This is a revised version of the [DelftX-Database](https://github.com/AngusGLChen/DelftX-Database) setup. While the number of steps are similar, they are now less ambiguous. `main.py` has been updated to Python 3 where necessary.

This setup takes you from a bunch of edX log files (and metadata files) to a fully working database that has the edX raw data nicely parsed and queryable.

**Note: use this code only for analyses conducted after the end of a MOOC; this is not suitable for incremental updates of the log files.** 

**Note II: this workflow requires Python 3!**

A Docker container is now used for the MySQL server to avoid a painful manual installation. The edx log data is loaded into the database. This requires extensive preprocessing. This is only necessary once. Once the container is running any MySQL client (command-line or GUI can access it). 

# Open Issue

The database resides within a Docker container, it is as of now not persistent, i.e. once the Docker container shuts down (terminal closes, machine shuts down, ...), the whole process has to be repeated. Alternatively, once the whole process is finished, an SQL dump can be made and persisted to local disk.
Should be fixed soon.


# Short vs. Long Version

The step-by-step guide below explains every step to some extent. A lot of manual copy and pasting is required. The short version is less explanatory but also quicker to execute. Decide for yourself which one you want to use.

## Step-by-step: Short Version

A. Install [Docker](https://www.docker.com/).

B. Open a terminal and navigate to an empty directory (lets call it `$MY_DIR$`) in which you want to store your data. Download the shell script:
```
curl -o steps-1-9.sh https://raw.githubusercontent.com/chauff/edx-analysis/master/steps-1-9.sh
```
The only change recommended to make to the contents of `steps-1-9.sh` is the course name (lines 4 and 5). By default it is `COURSE1` but that can be replaced by any reasonable identifier, e.g. `FP101x`. It is not necessary though. If you want several courses to process at once, add additional `mkdir course_log/COURSEX` and `mkdir course_log/COURSEX/metadata` entries to this file. If you only want to process data for a single MOOC no changes are necessary. Make the file executable and execute in the terminal:
```
chmod 777 steps-1-9.sh
./steps-1-9.sh
```
The result of this script is a running instance of a MySQL server with the database schema that we need for our edx log data. Keep the terminal open (otherwise the MySQL server will shut down).

C. Take a look at the file structure the script just created. Move the edx log data into the correct folders as described in steps 2 and 3 of the step-by-step long version below.

D. Continue with step 10 of the long version (that starts the process of adding the data to the database). Follow then step 11 to test whether everything works as expected. Instead of step 11 (command line interface), you can also open a graphical UI for MySQL such as SequelPro:
[Database parameters](sequelpro.png)

That's it.


## Step-by-step: Long Version
0. Install [Docker](https://www.docker.com/).

1. Open a terminal and navigate to an empty directory (lets call it `$MY_DIR$`) in which you want to store your data. Execute the following five commands (copy to terminal & Enter for each command) to create sub-directories and an empty file. Note that `COURSE1` can be replaced by any identifier of your choice, e.g. `FP101x`:
```bash
mkdir course_log
mkdir course_log/Daily_Logs
mkdir course_log/COURSE1/
mkdir course_log/COURSE1/metadata
touch course_log/translated_course_list
```

2. Populate the `course_log/Daily_Logs` sub-directory by copying the daily edx log files (`delftx-edx-events-201X-MM-DD.log.gz`) into it. Keep the files in the \*.gz format.

3. Populate the `course_log/COURSE1/metadata` sub-directory by copying all course metadata files into it (those files contain grades, user overviews, etc.). If this data is downloaded as a single archive (zip or tar) from e.g. surfsara, it needs to be uncompressed manually. 

4. Make sure the terminal's current directory is still `$MY_DIR$` (this can be checked with the command `pwd`) . Download the Python scripts that preprocess the daily log files with the following terminal commands:
```
mkdir translation
curl -o main.py https://raw.githubusercontent.com/chauff/edx-analysis/master/main.py
curl -o translation/ForumMode.py https://raw.githubusercontent.com/chauff/edx-analysis/master/translation/ForumMode.py
curl -o translation/Functions.py https://raw.githubusercontent.com/chauff/edx-analysis/master/translation/Functions.py
curl -o translation/LearnerMode.py https://raw.githubusercontent.com/chauff/edx-analysis/master/translation/LearnerMode.py
curl -o translation/QuizMode.py https://raw.githubusercontent.com/chauff/edx-analysis/master/translation/QuizMode.py
curl -o translation/SurveyMode.py https://raw.githubusercontent.com/chauff/edx-analysis/master/translation/SurveyMode.py
curl -o translation/VideoMode.py https://raw.githubusercontent.com/chauff/edx-analysis/master/translation/VideoMode.py
```

5. The `main.py` script writes data to the database; it needs information about how to access the DB. The current directory is still `$MY_DIR$`. Lets create a file `config` with the necessary information via the terminal:
```
touch config
echo "[mysqld]" >> config
echo "user = root" >> config
echo "password = 123456" >> config
echo "host = 127.0.0.1" >> config
echo "database = DelftX" >> config
echo "[data]" >> config
echo "path = $(pwd)/course_log/" >> config
echo "remove_filtered_logs = 0" >> config
```

6. Download the database schema. The terminal's current directory is still `$MY_DIR$`:
```
mkdir docker-container
curl -o docker-container/DelftX.sql https://raw.githubusercontent.com/AngusGLChen/DelftX-Database/master/DelftX.sql
```

7. It is time for the database. We use Docker, so all we need to do is create a Docker file and fill in the details. Still at `$MY_DIR$` in the terminal:
```
touch docker-container/Dockerfile
echo "FROM mysql" >> docker-container/Dockerfile
echo "ENV MYSQL_USER root_user" >> docker-container/Dockerfile 
echo "ENV MYSQL_ROOT_PASSWORD 123456" >> docker-container/Dockerfile
echo "ENV MYSQL_DATABASE DelftX" >> docker-container/Dockerfile
echo "ADD DelftX.sql /docker-entrypoint-initdb.d/" >> docker-container/Dockerfile
echo 'CMD ["mysqld"]' >> docker-container/Dockerfile
```

8. Still in `$MY_DIR$`, build the Docker image via the terminal command:
```
docker build docker-container/
```
The last line of output this command produces looks something like this: `Successfully built bf3a4f120a22`. We need this hash identifier for the next step.

9. Now start the database (replace the hash identifier here with the actual one you observed):
```
docker run -p 127.0.0.1:3306:3306 bf3a4f120a22
```
**Keep this terminal open**. If it is closed, the container (database server) ceases to exist. That's it, the MySQL server is now running. We can test this quickly by connecting the MySQL client from our **local machine** to the container, like so (here `mysql` is the MySQL command line client binary and we assume the binary is in the current directory - adapt the path to your own `mysql` installation) from a **new** terminal tab/window:
```
./mysql -h localhost -P 3306 --protocol=tcp -u root -p
```
Besides the command line client, MySQL's Workbench or another GUI-based application can of course also be used. 

10. Finally, we have to run the preprocessing script from a **new** terminal tab/window:
```
python main.py config
```
If it throws an error `No module mysql`, run `pip install mysql-connector-python-rf` to install [Connector/Python](https://dev.mysql.com/doc/connector-python/en/connector-python-installation.html). 

This will take quite a while to run, "All finished" indicates that everything went well.

11. To check whether the database has any content, the MySQL client can be used again (same as in step 9, via the terminal):
```
./mysql -h localhost -P 3306 --protocol=tcp -u root -p
```
You will be asked for the password, it is `123456` as given in the configuration file. The interactive `mysql` shell can be tested like this for instance:
```
show databases;
use DelftX;
show tables;
select * from submissions;
```

That's it. For completeness, the Docker container can be stopped nicely as follows: the command `docker ps` shows the list of containers available on the machine. Note the CONTAINER ID of each container. The command `docker stop [CONTAINER ID]` will stop the respective container. The container can be started again with the command in step 9 (it does not have to be rebuilt every time).

