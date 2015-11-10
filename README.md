# xlr-jython-code-snippets
XLR code snippets using python and jython API

** Get previous task**

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

** Get Gated tasks before a task**

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

**Find a task by title**

```
def findTaskByTitle(title):
  list = []
  for item in phase.tasks:
    if item.title == title:
      list.append(item)
  return list
```

