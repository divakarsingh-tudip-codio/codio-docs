---
title : Autograde
description: ""


---

This option provides two ways of autograding unit scores for each student. The grading field is populated by a script authored by the project author without the need to manually populate it. The autograding script is triggered once a unit is marked as complete.

A Unit is marked as complete in any of the following ways

1. Students mark the unit as complete from their dashboard.
1. The teacher can also mark the unit as complete for a student from the Classroom dashboard with the unit selected.
1. The teacher can mark all units as complete for all students by pressing the **Actions** button. This button appears on the unit screen.
1. If you are using the **Unit Duration** feature, all student units are marked as complete as soon as the unit duration expiry date and time is reached.

The two autograding options can be found in the unit settings.

1. Use the auto-graded assessments within the unit to auto-populate the grading field with the aggregate % score from all assessments.
1. Run a script to generate the grading either as soon as the student (or teacher) marks the unit as complete in their dashboard or when the [unit duration](/classes/unitmanagement/settings-info/unit-duration/) expires.

The two options can be found in the **AUTOGRADE METHOD** drop-down list.

<a name="transfer"></a>
# Transferring authored content assessment total
If you have created auto-graded assessments within your authored content, Codio aggregates all scores so you can see them in the Classroom dashboard. You will see that there is a total percentage calculated. This percentage value is transferred into the grading field. If you are using [LMS integration](/classes/lti) then this grading field is then transferred into your LMS gradebook once you [release the grades](/classes/monitor/grading).

<a name="script"></a>

# Running a custom script
A more advanced way of populating the grading field is to write your own custom script that evaluates the student code. This script can then transfer the grading value into the grading field.

If you are using an LMS platform with Codio then be sure to write a percentage value into this field to maintain compatibility with LMS gradebooks.

![authtoken](/img/grading-secure.png)

<a name="securescripts"></a>
# Secure scripts
If you want your scripts to run securely such that the student has no way of either viewing the script or viewing other files that might contain secure data then you should place those scripts and files in the `.guides/secure` folder. Codio ensures that only the original project author is able to access this folder but when it is assigned to Students as a Unit, it is not accessible in any way and the script runs in an ephemeral container isolated from the students unit.


# Timeout
Your script must execute within 3 minutes or a timeout error will occur.

# Accessing authored content assessment results
You are able to get scores attained by students in authored content based autograded assessments. This data is in JSON format and can be accessed from the `CODIO_AUTOGRADE_ENV` environment variable. Below is an example.

```
{
  "assessments": {
    "stats": {
      "total": 2,
      "answered": 2,
      "correct": 2,
      "totalPoints": 12,
      "points": 8
    },
    "info": [{
      "name": "Test 1",
      "points": 5,
      "answer": {
        "correct": true,
        "points": 5
      }
    }, {
      "name": "Test 2",
      "points": 7,
      "answer": {
        "correct": true,
        "points": 3
      }
    }]
  },
  "completedDate": "2017-02-07T09:47:54.471Z"
}
```

You can get both summary data and data for each assessment individually.


<a name="regrading"></a>
# Regrading for an individual student
If students set their work to 'complete' such that an autograde step is triggered then you can regrade the work by resetting the complete switch and then setting it again, which re-triggers the autograding.

# Regrading all students
From the **Actions** area of the Unit, you can regrade all students that have already been auto-graded by pressing the **Regrade All** button. This is useful if you have found a bug in your grading script. If you follow (or use) the code sample shown at the bottom of this page you can see how the original student submission date is handled.

# Testing and debugging your grading scripts
**IMPORTANT**: please read this section carefully.

We provide a way of testing autograding scripts when authoring your project. This is described below. You should make use of this before publishing your project to a class.

You should be aware that once the Unit has been published to the class, any changes made to the Unit's source project are not automatically reflected in the published Unit. As a result, if you include your main grading logic within the project itself and if that script has bugs, you will not be able to fix the bugs without deleting the Unit, fixing the bug and finally republishing the Unit. All student data will be lost as a result. However, if all your scripts are stored in `.guides/secure` folder, you can update and test them and you can then [Update Unit](/classes/unitmanagement/settings-info/updateunit/)

Another strategy is to use a simple bootstrap launcher that loads and executes the script from a remote location that you can edit and debug independently of the Codio box.

The following example bash script shows a Python script that is located as a Gist on GitHub. This script might be called `.guides/secure/launcher.sh`.

```
#!/bin/bash
URL="https://gist.githubusercontent.com/MaximKraev/11cd4e43b0c43f79d9478efbe21ba1b9/raw/validate.py"
curl -fsSL $URL | python - $@
```
It is important that it is located in the `.guides/secure` folder. You then specify the full filepath `.guides/secure/launcher.sh` in the **Set custom script path** field in the Unit settings.

You are now free to debug the Python script and fix any bugs that you may have noticed once students have started work on the Unit.

## Testing your script in the IDE
We provide the ability to test your autograding script from the **Education -> Test Autograde Script** menu.

This option lets you specify the location to your autograding script and run it against the current project contents. It also lets you simulate scores attained by any autograded assessments located within the Codio Guide.

![authtoken](/img/autograde-test.png)

You should be aware of the following points.

- When you press the **Test Script** button
  - all output to `stdout` and `stderr` are displayed within the dialog
  - the grade as returned by your test script is at the bottom of the output section
- `stdout` and `stderr` output is not available when running for real (not in this test mode) as the autograding script runs invisibly when the Unit is marked as complete. As such, you should generate output for testing and debugging purposes only.
- If you want your script to provide any feedback to the student, then you should output it to a file that the student can access when opening the project at a later date. In this case you will need to allow read-only access to the project from the Unit settings after being marked as complete.
- Your script must execute within 3 minutes to avoid a timeout error.

If the Guide has autograded assessments then the test takes its data from the fields shown in the dialog. All of your assessment settings are accessed as described above under **Accessing Guide assessment results**.

# Example Python grading script
Below is an example Python file that might be loaded by the bootstrap script above.

Notice that the only code you need to modify is near the bottom. The other functions are helpers and can be used for any test in any Unit.

```python
import os
import random
import requests
import json
import datetime

# import grade submit function
import sys
sys.path.append('/usr/share/codio/assessments')
from lib.grade import send_grade

##################
# Helper functions #
##################


# Get the url to send the results to
CODIO_AUTOGRADE_URL = os.environ["CODIO_AUTOGRADE_URL"]
CODIO_UNIT_DATA = os.environ["CODIO_AUTOGRADE_ENV"]

def main():
  # Execute the test on the student's code
  grade = validate_code()
  # Send the grade back to Codio with the penatly factor applied
  res = send_grade(int(round(grade)))
  exit( 0 if res else 1)

########################################
# You only need to modify the code below #
########################################

# Your actual test logic
# Our demo function is just generating some random score
def validate_code():
  return random.randint(10, 100)

main()
```

<a name="examplebashscript"></a>
# Example Bash grading script

Below is an example bash script file that would be stored  in .guides/secure folder

```
#!/bin/bash
set -e
# Your actual test logic
# Our demo function is just generating some random score
POINTS=$(( ( RANDOM % 100 )  + 1 ))
# Show json based passed environment
echo $CODIO_AUTOGRADE_ENV
# Send the grade back to Codio
curl --retry 3 -s "$CODIO_AUTOGRADE_URL&grade=$POINTS"
```



