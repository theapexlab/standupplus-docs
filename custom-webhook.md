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
[...]
[funFact]
-> [customText]
[...]
```

## Example use cases:

- [Day of the year](#day-of-the-year)
- [Number of open PRs - Github](#number-of-open-prs-github)
- [Remaining days in current sprint - Jira](#remaining-days-in-current-sprint-jira)

## Pipedream

[Pipedream](https://pipedream.com/) is a low code integration platform for developers that allows you to connect APIs remarkably fast.

### Number of open PRs - GitHub

1. Select HTTP API as trigger.
2. With the plus button add a new step.
3. Select `GitHub` from the integrations.
4. Select the `Search Issues Or Pull Requests` action.
5. Connect your Github account, or select and already connected one.
6. For the query input the following `repo:[repo] is:pr is:open draft:false` where [repo] should be replaced by the name of your repository.
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

### Day of the year

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
