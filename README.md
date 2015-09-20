# Getting-and-cleaning-data-project
In this part i will explain how does the code work.
Initially, we must include all the necessary packages.
```
library(downloader)
library(data.table)
library(dplyr)
library(reshape2)
```
Next, we download the data from the given URL:
```
URL <- "https://d396qusza40orc.cloudfront.net/getdata%2Fprojectfiles%2FUCI%20HAR%20Dataset.zip"
download.file(URL,"E:/FER i posao/Coursera_Iversity/Data Scientist Specialization/3 Getting and cleaning data/Course project/dataset.zip",mode="wb")
```
In next step, i set the working directory to directory where i downloaded the data, and i unzipped it, since it is given in .zip format.
```
setwd("E:/FER i posao/Coursera_Iversity/Data Scientist Specialization/3 Getting and cleaning data/Course project")
unzip("dataset.zip",exdir="ProjectData")
```
Now we need to read the test and train data. Before reading them it is neccessary to navigate to the correct directory where those files are located.
```
setwd("./ProjectData")
setwd("./UCI HAR Dataset")
setwd("./test")
subject_test <- read.table("subject_test.txt")
X_test <- read.table("X_test.txt")
Y_test <- read.table("Y_test.txt")
setwd("..")
setwd("./train")
subject_train <- read.table("subject_train.txt")
X_train <- read.table("X_train.txt")
Y_train <- read.table("Y_train.txt")
```
Once we have our train and text data read into R, we need to bind together pairs. We will do that using rbind function.
```
Xbind <- rbind(X_test,X_train)
Ybind <- rbind(Y_test,Y_train)
Subbind <- rbind(subject_test,subject_train)
```
Next we adjust the column names in variables Subbind and Ybind (further meaning of each variable can be found in Codebook.md file)
```
setnames(Subbind,"V1","subject")
setnames(Ybind,"V1","activityNum")
```
Now we have everything ready for getting our final data. First we bind subject and activity data by columns, and then we add them to the test/train data. The final line turns the data.frame to data.table, so we could use functions from data.table package later in the code.
```
helpbind <- cbind(Ybind,Subbind)
finaldata <- cbind(Xbind,helpbind)
finaldata <- data.table(finaldata)
```
First we want to order the data in our final data by using the function setkey from the data.table package.
```
setkey(finaldata,subject,activityNum)
```
In the next part we will need to reshape our finaldata to include the feature labels. First we need to read in the features.txt into R and of course turn them into data.table format.
```
setwd("..")
features <- read.table("features.txt")
features <- data.table(features)
```
Next we set names V1 and V2 in features variable accordingly.
```
setnames(features,names(features),c("featureID","featureName"))
```
In the next part, we want to extract only the measurements which contain the word "mean" or the word "std". We can use the function grepl from the base R package to do that. This function returns logical vector, where we will have the value TRUE in rows which contain the word "mean" or the word "std" in their feature name. We can then use this logical vector to do the subsetting.
```
features <- features[grepl("mean|std",featureName)]
```
We are going to use the features variable to subset the wanted rows from our finaldata variable. To do that we need to have common columns. That's why we will add the featureCode variable (using the paste function).
```
features$featureCode <- features[,paste0("V",featureID)]
```
The next step is creation of the variable subset_key, which is in effect combination of the subject number, activity number, and V1 to V561 variables.
```
subset_key <- c(key(finaldata),features$featureCode)
```
Now we can use this key to subset only those rows from finaldata, that correspond to rows that only contain "mean" and "std" in their name (we use with=FALSE here to perform this operation faster).
```
finaldata <- finaldata [,subset_key,with=FALSE]
```
The only thing remaining now is reading the activites labels into R and reshaping data into our final tidy data. First we do the standard procedure of reading from txt file. We can also use setnames to set the column names in the activites variable. 
```
activities <- read.table("activity_labels.txt")
setnames(activities,"V1","activityNum")
setnames(activities,"V2","activityName")
```
Now we can merge the finaldata with the activities variable based on the activity number.
```
finaldata <- merge(finaldata,activities,by="activityNum")
```
And also reorder the data in the wanted order.
```
setkey(finaldata,subject,activityNum,activityName)
```
In the next part we will use the function melt from the reshape2 package. This function creates and unique id-variable combination in each row. We pass all the key variables from the finaldata as id variables, and we pass the featureCode as the variable into which we store the measured variable names.
```
finaldata <- data.table(melt(finaldata,key(finaldata),variable.name ="featureCode"))
```
Now we once again merge the finaldata with the features by featureCode.
```
finaldata <- merge(finaldata, features[,list(featureID, featureCode, featureName)], by="featureCode", all.x=TRUE)
```
The final step is getting the mean value of every V column. We can do that by combining function pair group_by/summarise (or summarise_each).
```
grouped <- group_by(finaldata, subject, activityNum)
mean_table <- summarise(grouped, mean=mean(value))
```
Since i grouped only by subject and activity number, i lost all other columns. That's why i need to add them again to get the complete tidy data. I can do that by merge and cbind.
```
mean_table <- merge(mean_table,activities,by="activityNum")
mean_table <- cbind(mean_table,features)
```
Finally, it only remains to write the table into txt format.
```
write.table(mean_table,"tidy_data.txt",row.names = TRUE)
```
