# Calculstore
// calculate a virtual InnerSource score from stars, watches, commits, and issues
function calculateScore(repo) {
    // initial score is 50 to give active repos with low GitHub KPIs (forks, watchers, stars) a better starting point
    let iScore = 50;
    // weighting: forks and watches count most, then stars, add some little score for open issues, too
    iScore += repo.forks_count * 5;
    iScore += (repo.subscribers_count ? repo.subscribers_count : 0);
    iScore += repo.stargazers_count / 3;
    iScore += repo.open_issues_count / 5;

    // updated in last 3 months: adds a bonus multiplier between 0..1 to overall score (1 = updated today, 0 = updated more than 100 days ago)
    let iDaysSinceLastUpdate = (new Date().getTime() - new Date(repo.updated_at).getTime()) / 1000 / 86400;
    iScore = iScore * ((1 + (100 - Math.min(iDaysSinceLastUpdate, 100))) / 100);

    // evaluate participation stats for the previous  3 months
    repo._InnerSourceMetadata = repo._InnerSourceMetadata || {};
    if (repo._InnerSourceMetadata.participation) {
        // average commits: adds a bonus multiplier between 0..1 to overall score (1 = >10 commits per week, 0 = less than 3 commits per week)
        let iAverageCommitsPerWeek = repo._InnerSourceMetadata.participation.slice(-13).reduce((a, b) => a + b) / 13;
        iScore = iScore * ((1 + (Math.min(Math.max(iAverageCommitsPerWeek - 3, 0), 7))) / 7);
    }

    // boost calculation:
    // all repositories updated in the previous year will receive a boost of maximum 1000 declining by days since last update
    let iBoost = (1000 - Math.min(iDaysSinceLastUpdate, 365) * 2.74);
    // gradually scale down boost according to repository creation date to mix with "real" engagement stats
    let iDaysSinceCreation = (new Date().getTime() - new Date(repo.created_at).getTime()) / 1000 / 86400;
    iBoost *= (365 - Math.min(iDaysSinceCreation, 365)) / 365;
    // add boost to score
    iScore += iBoost;
    // give projects with a meaningful description a static boost of 50
    iScore += (repo.description?.length > 30 || repo._InnerSourceMetadata.motivation?.length > 30 ? 50 : 0);
    // give projects with contribution guidelines (CONTRIBUTING.md) file a static boost of 100
    iScore += (repo._InnerSourceMetadata.guidelines ? 100 : 0);
    // build in a logarithmic scale for very active projects (open ended but stabilizing around 5000)
    if (iScore > 3000) {
        iScore = 3000 + Math.log(iScore) * 100;
    }
    // final score is a rounded value starting from 0 (subtract the initial value)
    iScore = Math.round(iScore - 50);
    // add score to metadata on the fly
    repo._InnerSourceMetadata.score = iScore;

    return iScore;
}
