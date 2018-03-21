I'm trying to find the strength of the association between high rushing yards and wins. I'm placed the abbreviation of every team in a list, and I am attempting to iterate through the list and execute certain code for each team individually, then place it into an array. I have an array that will be filled with 'Y' (over 150 rush yards) or 'N' (less than 150 rush yards) values for every game last season and an array with 'W', 'T', or 'L' values for every game last season. Here is what I have. I managed to get two arrays: one filled with 'W's, 'T's, and 'L's and one filled with 'Y's and 'N's. The problem is I when I checked to see how many items were in each array (it should be 256), the length of winLoss was 233 and the length of over150 was 512.

    import nfldb

    db = nfldb.connect()

    over150 = []
    winLoss = []

    counter = 0
    counter2 = 0

    teamRushYds = 0

    teamsList = ['ARZ','STL','ATL','BAL','BUF','CAR','CHI','CIN','CLE','DAL','DEN','DET','GB','KC','HOU','IND','JAC','MIA','MIN','NE','NO','NYG','NYJ','OAK','PHI','PIT','SD','SF','SEA','TB','TEN','WAS']

    for t in teamsList:
        for i in xrange(16):
            q = nfldb.Query(db)
            q.game(season_year=2015,season_type='Regular',week=i+1,home_team=t)
            for g in q.as_games():
                if g.home_score > g.away_score:
                    winLoss.append('W')
                elif g.home_score < g.away_score:
                    winLoss.append('L')
                else:
                    winLoss.append('T')
            q = nfldb.Query(db)
            q.game(season_year=2015,season_type='Regular',week=i+1,home_team=t)
            q.player(position='RB',team=t)
            for a in q.as_aggregate():
                teamRushYds += a.rushing_yds
            if teamRushYds > 150:
                over150.append('Y')
            else:
                over150.append('N')
            teamRushYds = 0

    print len(winLoss)
    print len(over150)

I am just testing to see if the winLoss and over150 arrays have the correct amount of elements, which should be 256 (256 total games in regular season. Any help is appreciated.

Thanks
