# xlr-jython-code-snippets
XLR code snippets using python and jython API

### Skip the current task (Uses OnFailureHandler (8.1+)

```
task.setTaskFailureHandlerEnabled(True)
task.setTaskRecoverOp(TaskRecoverOp.SKIP_TASK)
taskApi.updateTask(task)
raise Exception("Jenkins job was not built.")
```

### Set the release title using release variables

```
#use the 'component' and 'version' release varible as title 
title = "{component}/{version}".format(**releaseVariables)
print(title)
release.title = title
releaseApi.updateRelease(release)
```

### Get previous task

```
def previousTask(task):
  index = 0
  for item in phase.tasks:
    if item.id == task.id:
      break
    index = index + 1
  return task.getPhase().tasks[index-1]
print task.title
print "previous task is: " + str(previousTask(task).title)
```

### Get Gated tasks before a task

```
def gatesBeforeTask(task):
  gatesList = []
  for item in phase.tasks:
    if str(item.getTaskType()) == "xlrelease.GateTask":
     gatesList.append(item)
    if item.id == task.id:
     break
  return gatesList

print "start gates " + str(phase.getStartGates())
for item in gatesBeforeTask(task):
  print item.title
```

### Find a task by title

```
def findTaskByTitle(title):
  list = []
  for item in phase.tasks:
    if item.title == title:
      list.append(item)
  return list
```
### Check/uncheck a gate condition via REST API

\# Sample call:  python gateupdate.py Release546450 UAT 'Gate 2' 'Gate 2 condition 1' 'true' 

```
import requests
import sys

releaseId = sys.argv[1]
phaseTitle = sys.argv[2]
gateTitle = sys.argv[3]
conditionTitle = sys.argv[4]
conditionChecked = sys.argv[5]

release = requests.get('http://admin:xlradmin@localhost:5516/releases/' + releaseId)
for phase in release.json()['phases']:
  if phase['title'] == phaseTitle:
    for task in phase['tasks']:
      if task['title'] == gateTitle:
        for condition in task['conditions']:
          if condition['title'] == conditionTitle:
            condition['checked'] = conditionChecked
            r = requests.put('http://admin:xlradmin@localhost:5516/gates/conditions/' + condition['id'], json=condition)
            print r.status_code	
```


### Fetch variables from another release

```
rvar = releaseApi.getVariables(releaseidvar)
for r in rvar: 
    print "key:%s , value:%s \n" % (r._delegate.key,r._delegate.value)
```
or

```
myRelease = getCurrentRelease()
for r in myRelease.variables: 
   print "key:%s , value:%s \n" % (r.key, r.value)

```

### Update automated task server entry

```
t = taskApi.searchTasksByTitle("buildjar","Build",release.id)[0]
jenkinslist = configurationApi.searchByTypeAndTitle("jenkins.Server",releaseVariables['jenkinsserver'])
t.pythonScript.jenkinsServer = jenkinslist[0]
taskApi.updateTask(t)
```

### Fetch URL for selected server in a automated task

```
t = taskApi.searchTasksByTitle("buildjar","Build",release.id)
jenkinsUrl = t[0].pythonScript.jenkinsServer.url
releaseVariables['jarurl'] = "%s/job/buildjar/%s/artifact/abc.jar" % (jenkinsUrl, releaseVariables['buildjarbn'])
```

### Create table/fancy markdown in a task description

```
a = [1,2,3,4,5,6]
out=  "|Heading \n |---\n"
for item in a:
    out= "{0}|{1}\n".format(out,item)
task.description = out
taskApi.updateTask(task)
```
#### Image
![images/tableintaskdescription.png](images/tableintaskdescription.png)



## Xlrelease add dynamic tags

```
release.tags.add('abc')
releaseApi.updateRelease(release)
```



## Print classpath 

```
from java.lang import ClassLoader
cl = ClassLoader.getSystemClassLoader()
paths = map(lambda url: url.getFile(), cl.getURLs())
print paths
```

## Add a folder

```
from com.xebialabs.xlrelease.domain.folder import Folder
newFolder = Folder()
newFolder.title="MyFolder"
folderApi.addFolder("Applications",newFolder)
```

## Copy a template

```
templates = folderApi.getTemplates("Applications")
# Pick the first template 
template = templates[0]
# Give it a new title
template.title = "new template"
templateApi.createTemplate(template)
```

## Generate a release pipeline dynamically with parallel group and tasks from a provided dataset

```
import json
teamProjectName = "MyProject"
buildDefinitionName = "mybuildDef"
datamap = '''[
	{
		"name": "libs",
		"entries": [
			{
				"name": "a-library",
				"skip": "false"
			},
			{
				"name": "b-library",
				"skip": "false"
			},
			{
				"name": "c-library",
				"skip": "true"
			}
		],
		"execution": "sequence"
	},
	{
		"name": "messages",
		"entries": [
			{
				"name": "a-messages",
				"skip": "false"
			},
			{
				"name": "b-messages",
				"skip": "true"
			},
			{
				"name": "c-messages",
				"skip": "false"
			}
		],
		"execution": "parallel"
	},
	{
		"name": "services",
		"entries": [
			{
				"name": "a-service",
				"skip": "false"
			},
			{
				"name": "b-service",
				"skip": "false"
			}
		],
		"execution": "parallel"
	}
]
'''
dataobj = json.loads(datamap)

for item in dataobj:
    phase = phaseApi.newPhase(item['name'])
    phase = phaseApi.addPhase(release.id, phase)
    if str(item['execution']) == "parallel":
        pgrouptask = taskApi.newTask("xlrelease.ParallelGroup")
        pgrouptask.title = "Parallel Run"
        phase = taskApi.addTask(phase.id, pgrouptask)
    for entry in item['entries']:
        task = taskApi.newTask("vsts.QueueBuild")
        task.title = entry['name']

        task.pythonScript.setProperty("teamProjectName", teamProjectName)
        task.pythonScript.setProperty("buildDefinitionName", buildDefinitionName)
        
        if entry['skip'] == "true":
            task.precondition = "True == False"
        taskApi.addTask(phase.id, task)
        
 ```

## Add a Custom task (MyCustomTask.TestMapOutPut) in curent running Release and set a Key-Value (dicoPackageMap) in output property (MapOut)

```
curRelease = getCurrentRelease()
curPhase = getCurrentPhase().title
task = taskApi.newTask("MyCustomTask.TestMapOutPut")
task.title = "ScriptCreated"
task.variableMapping = {"pythonScript.MapOut" : "$""{dicoPackageMap}"}
createdTask=taskApi.addTask(getCurrentPhase().id, task)
taskApi.assignTask(createdTask.id,"Admin")

```

## Adding Date, working with Date

```
from java.util import Calendar, Date
cal = Calendar.getInstance()
cal.setTime(Date())
cal.add(Calendar.DAY_OF_WEEK, 7)
print cal.getTime()

```

## Add a Task as the last Task in phase by copying a previous Task

```
t = getTasksByTitle("t1", "New Phase")[0]
newtask = taskApi.copyTask(t.id, phase.id, len(phase.getTasks()))


```

## Scheduling a task dynamically

```
from java.util import Calendar, Date
cal = Calendar.getInstance()
cal.setTime(getCurrentTask().startDate)
cal.add(Calendar.DAY_OF_WEEK, 2)
# getting a task that is ahead of current task in the flow
t = getTasksByTitle("t1", "New Phase")[0]
t.setScheduledStartDate(cal.getTime()) 
taskApi.updateTask(t)

```

## Create a Self Scheduling Task Block - Style 1
This example makes use of a Sequential Group. You can have any number of tasks within the sequential group but the last task should be a Jython Script/Custom Task with the following Script.  The script will replicate its parent Sequential Group and then reschedule the newly cloned group for a certain datetime

```
parenttask = getCurrentTask().container
# find the sequential group child
sequentialGroupId = None
activetask = None
# Iterating on the children to find SequentialGroup ID and the last active manual task ID
for child in parenttask.getChildren():
if str(child.type).lower() == "xlrelease.SequentialGroup".lower():
    sequentialGroupId = child.id
if str(child.type).lower() == "xlrelease.Task".lower() and str(child.status).lower() == "IN_PROGRESS".lower():
    activetask = child.id

# Setting up a notification task
notfication = taskApi.newTask("xlrelease.NotificationTask")
notfication.subject = "Sending Notification at %s" % datetime.today()
notfication.addresses = Set(["a@gmail.com", "b@gmail.com"])
notfication.body = "hi Hello"
taskApi.addTask(sequentialGroupId, notfication)

# Setting up a manual task
manualtask = taskApi.newTask("xlrelease.Task")
manualtask.title = "Waiting for Activity on %s" % ( datetime.today() + timedelta(days=1) )
taskApi.addTask(sequentialGroupId, manualtask)

# assigning it to the admin user to mark it complete in the next step
taskApi.assignTask(activetask,"admin")
taskApi.completeTask(activetask,Comment())

```

## Create a Self Scheduling Task Block - Style 2 ( Better )

```
from java.util import Calendar, Date
import time

# Get the parent block for current script task
parenttask = getCurrentTask().container

# Use calendar to increment the time
cal = Calendar.getInstance()
cal.setTime(parenttask.startDate)
cal.add(Calendar.DAY_OF_WEEK, 2)

# Create a new task from that parenttask as the next task 
newtask = taskApi.copyTask(parenttask.id, phase.id, len(phase.getTasks()))

# Set the newtask to the new time and title
newtask.setScheduledStartDate(cal.getTime()) 
newtask.title = parenttask.title

# Update the new task
taskApi.updateTask(newtask)

```

## Create/Populate a List Variable using Jython
There are different interpreted types for different type of variables within the api. Usual way of knowing the right name is eg.
If variable type is **StringVariable**, then xlr type is **xlrelease.StringVariable**

Some variable types are 
* StringVariable
* ListStringVariable
* MapStringStringVariable
* PasswordStringVariable
* SetStringVariable

```
from com.xebialabs.xlrelease.api.v1.forms import Variable  
mylist = [1,2,3,4]
listvar = Variable()
listvar.value = mylist
listvar.key = "mylistvar"
listvar.type = "xlrelease.ListStringVariable"
releaseApi.createVariable(release.id,listvar )

```




