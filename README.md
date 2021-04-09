# Team-Young
Daily Grind List
The "Daily Grind List" puts all of your daily/weekly/monthly and overall goals on an application with a spot to check off those goals. 
Software: Visual Studio Code 
Availability: All major phone carriers and devices. 



Python Code Reference: const checkOutstandingTasks = require('./src/check-outstanding-tasks');

module.exports = (app) => {
  app.log('Yay! The app was loaded!');

  // watch for pull requests & their changes
  app.on([
    'pull_request.opened',
    'pull_request.edited',
    'pull_request.synchronize',
    'issue_comment', // for comments on github issues
    'pull_request_review', // reviews
    'pull_request_review_comment', // comment lines on diffs for reviews
  ], async context => {
    const startTime = (new Date).toISOString();

    // lookup the pr
    let pr = context.payload.pull_request;

    // check if this is an issue rather than pull event
    if (context.event == 'issue_comment' && ! pr) {
      // if so we need to make sure this is for a PR only
      if (! context.payload.issue.pull_request) {
        return;
      }
      // & lookup the PR it's for to continue
      let response = await context.github.pulls.get(context.repo({
        pull_number: context.payload.issue.number
      }));
      pr = response.data;
    }

    let outstandingTasks = checkOutstandingTasks(pr.body);

    // lookup comments on the PR
    let comments = await context.github.issues.listComments(context.repo({
      issue_number: pr.number
    }));

    // as well as review comments
    let reviewComments = await context.github.pulls.listReviews(context.repo({
      pull_number: pr.number
    }));
    if (reviewComments.data.length) {
      comments.data = comments.data.concat(reviewComments.data);
    }

    // and diff level comments on reviews
    let reviewDiffComments = await context.github.pulls.listComments(context.repo({
      pull_number: pr.number
    }));
    if (reviewDiffComments.data.length) {
      comments.data = comments.data.concat(reviewDiffComments.data);
    }

    // & check them for tasks
    if (comments.data.length) {
      comments.data.forEach(function (comment) {
        let commentOutstandingTasks = checkOutstandingTasks(comment.body);
        outstandingTasks.total += commentOutstandingTasks.total;
        outstandingTasks.remaining += commentOutstandingTasks.remaining;
      });
    }

    let check = {
      name: 'task-list-completed',
      head_sha: pr.head.sha,
      started_at: startTime,
      status: 'in_progress',
      output: {
        title: (outstandingTasks.total - outstandingTasks.remaining) + ' / ' + outstandingTasks.total + ' tasks completed',
        summary: outstandingTasks.remaining + ' task' + (outstandingTasks.remaining > 1 ? 's' : '') + ' still to be completed',
        text: 'We check if any task lists need completing before you can merge this PR'
      }
    };

    // all finished?
    if (outstandingTasks.remaining === 0) {
      check.status = 'completed';
      check.conclusion = 'success';
      check.completed_at = (new Date).toISOString();
      check.output.summary = 'All tasks have been completed';
    };

    // send check back to GitHub
    return context.github.checks.create(context.repo(check));
  });
};



also:
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <!-- <script type="module" src="../dist/index.js"></script> -->
  <script type="module" src="https://unpkg.com/@github/task-lists-element@latest"></script>
  <title>task-lists-element demo</title>
</head>
<body>
  <task-lists sortable>
    <ul class="contains-task-list">
      <li class="task-list-item">
        <label>
          <input type="checkbox" class="task-list-item-checkbox">
          Hubot
        </label>
      </li>
      <li class="task-list-item">
        <label>
          <input type="checkbox" class="task-list-item-checkbox">
          Bender
        </label>
      </li>
    </ul>

    <ul>
      <li>
        Nested

        <ul class="contains-task-list">
          <li class="task-list-item">
            <label>
              <input type="checkbox" class="task-list-item-checkbox">
              WALL-E
            </label>
          </li>
          <li class="task-list-item">
            <label>
              <input type="checkbox" class="task-list-item-checkbox">
              R2-D2
            </label>

            <ul class="contains-task-list">
              <li class="task-list-item">
                <label>
                  <input type="checkbox" class="task-list-item-checkbox">
                  Baymax
                </label>
              </li>
            </ul>
          </li>
          <li class="task-list-item">
            <label>
              <input type="checkbox" class="task-list-item-checkbox">
              BB-8
            </label>
          </li>
        </ul>
      </li>
    </ul>
  </task-lists>
  <pre class="events"></pre>
  <script type="text/javascript">
    const events = document.querySelector('.events')
    document.addEventListener('task-lists-check', function(event) {
      events.append(`task-lists-check - checked: ${event.detail.checked}, position: ${event.detail.position}\n`)
    })

    document.addEventListener('task-lists-move', function(event) {
      events.append(`task-lists-move - from: ${event.detail.src}, to: ${event.detail.dst}\n`)
    })
  </script>
</body>
</html>

Additonal References: https://github.com/s-young68/github-task-list-completed
