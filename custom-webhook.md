# Custom webhook handling

You can customize Olivia's initial speech by providing a webhook url in the Google Calendar event's description:

Format:

`standupplus_url: [webhook's url]`

The webhook MUST return a JSON object with the following structure:

```js
{
  override: Boolean,
  customText: String
}
```

If `override` is true, the `customText` value will be used instead of the default speech.

Otherwise, `customText` will be injected into the default speech, in the following manner:

```
Happy ${day}!
Before we start let me share that ${funFact}. I hope this highlights your day. But let's get serious.
${customText}
${openAiGeneratedStarterMessage}
```

## Example use cases:

- [Day of the year](#day-of-the-year)
- [Number of open PRs - Github](#number-of-open-prs---github)
- [Remaining days in current sprint - Jira](#remaining-days-in-current-sprint---jira)

## Pipedream

[Pipedream](https://pipedream.com/) is a low code integration platform for developers that allows you to connect APIs remarkably fast.

We provided a premade Pipedream workflow for each of the above use cases, and also the steps to recreate them.

### Day of the year

[Premade Pipedream workflow](https://pipedream.com/@apexedit/custom-webhook-day-of-the-year-override-false-p_BjCAleD)

Steps to recreate:

1. Select HTTP API as trigger.
2. With the plus button add a new step.
3. Select `Run Node.js code` from the right panel.
4. Copy the following code and paste it into the `code` input:

```js
async (event, steps) => {
  const now = new Date();
  const start = new Date(now.getFullYear(), 0, 0);
  const diff =
    now -
    start +
    (start.getTimezoneOffset() - now.getTimezoneOffset()) * 60 * 1000;
  const oneDay = 1000 * 60 * 60 * 24;
  const numberOfDay = Math.floor(diff / oneDay);

  const body = {
    override: true,
    customText: `This is the ${numberOfDay}th day of the year.`,
  };

  $respond({
    status: 200,
    body,
  });
};
```

### Number of open PRs - GitHub

[Premade Pipedream workflow](https://pipedream.com/@apexedit/custom-webhook-github-integration-example-p_aNCqAVb)

Steps to recreate:

1. Select HTTP API as trigger.
2. With the plus button add a new step.
3. Select `GitHub` from the integrations.
4. Select the `Search Issues Or Pull Requests` action.
5. Connect your Github account, or select and already connected one.
6. For the `query` input add the following `repo:[repo] is:pr is:open draft:false` where [repo] should be replaced by the name of your repository.
7. Add a new step.
8. Select `Run Node.js code` from the right panel.
9. Copy the following code and paste it into the `code` input:

```js
async (event, steps) => {
  const numberOfOpenPullRequests =
    steps.github_search_issues_pull_requests.$return_value.total_count || 0;

  const customText =
    numberOfOpenPullRequests === 0
      ? "There is no open pull request waiting for a review. Nice job guys, keep up this speed!"
      : `Currently, there are ${numberOfOpenPullRequests} open pull requests waiting to be reviewed and merged. 
    Let's finish them first, then jump to the next task.`;

  $respond({
    status: 200,
    body: {
      override: true,
      customText,
    },
  });
};
```

### Remaining days in current sprint - Jira

[Premade Pipedream workflow](https://pipedream.com/@apexedit/custom-webhook-jira-integration-example-p_xMCxeep)

Steps to recreate:

1. Select HTTP API as trigger.
2. With the plus button add a new step.
3. Select `Jira` from the integrations.
4. Select the `Get Projects Paginated` action.
5. Connect your Jira account, or select and already connected one.
6. Copy the following code and paste it into the `code` input, providing your project id for the JIRA_PROJECT_ID variable:

```js
async (event, steps) => {
  // Change this to the relevant JIRA project
  const JIRA_PROJECT_ID = "";

  // First we must make a request to get our the cloud instance ID tied
  // to our connected account, which allows us to construct the correct REST API URL.
  const resp = await require("@pipedreamhq/platform").axios(this, {
    url: `https://api.atlassian.com/oauth/token/accessible-resources`,
    headers: {
      Authorization: `Bearer ${auths.jira.oauth_access_token}`,
    },
  });

  // Assumes the access token has access to be a single instance
  const cloudId = resp[0].id;

  // Construct querry with JQL syntax then encode it and us it as querry param
  const jql = encodeURIComponent(`project = ${JIRA_PROJECT_ID}`);

  // This will return issues related to the JIRA project
  return await require("@pipedreamhq/platform").axios(this, {
    url: `https://api.atlassian.com/ex/jira/${cloudId}/rest/api/3/search?jql=${jql}`,
    headers: {
      Authorization: `Bearer ${auths.jira.oauth_access_token}`,
      Accept: "application/json",
    },
  });
};
```

7. Add a new step.
8. Select `Run Node.js code` from the right panel.
9. Copy the following code and paste it into the `code` input, providing FIELD_ID.

```js
async (event, steps) => {
  const differenceInBusinessDays = require('date-fns/differenceInBusinessDays')

  // Add custom field id here
  // this will be extracted from previous step's result
  // Sprint custrom field id:  customfield_10020
  const FIELD_ID = "customfield_10020"

  const now = new Date()

  const sprints = steps.jira.$return_value.issues.map(({ fields }) => fields[FIELD_ID]).filter(sprint => !!sprint).flat()
  const activeSprint = sprints.find(sprint => sprint && sprint.state === 'active' && new Date(sprint.endDate) > now)

  const diffDays = differenceInBusinessDays(new Date(activeSprint.endDate), now)

  const body = {
    override: false,
    customText: `We have ${diffDays} ${diffDays === 1 ? 'day' : 'days'} left in this sprint.`
  }

  $respond({
    status: 200,
    body
  })
};
```
