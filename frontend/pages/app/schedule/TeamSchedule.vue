<template>
  <div class="team-schedule">
    <q-page class="q-pa-md">
      <q-card>
        <q-card-section>
          <div class="text-h6">
            <img :src="teamImage" alt="Team Logo" class="team-logo" />
            {{ teamName }}'s Schedule
          </div>
        </q-card-section>
        <q-card-section>
          <vue-cal
            :events="events"
            :disable-views="['years', 'year', 'month']"
            default-view="week"
            style="height: 600px"
            @range-change="onRangeChange"
          />
        </q-card-section>
      </q-card>
    </q-page>
  </div>
</template>

<script>
import VueCal from "vue-cal";
import "vue-cal/dist/vuecal.css";
import { format, subMonths, addMonths } from "date-fns";

export default {
  name: "TeamSchedule",
  components: {
    VueCal,
  },
  data() {
    return {
      teamId: "",
      teamName: "",
      teamImage: "", // New data property for team image
      schedule: [],
      events: [],
      currentMonth: format(new Date(), "yyyy-MM"),
      seasonId: "", 
      leagueId: "", 
    };
  },
  created() {
    this.fetchInitialData();
  },
  methods: {
    async fetchInitialData() {
      try {
        // Fetch user team information
        const userTeamResponse = await fetch(
          `https://api.powerplay.com/users/me/team`
        );
        const userTeamData = await userTeamResponse.json();
        /* The response should look like this:
          {
            "teamId": "12345",
            "teamName": "Team Awesome",
            "teamImage": "https://example.com/path/to/team-logo.png"
          }
        */
        this.teamId = userTeamData.teamId;
        this.teamName = userTeamData.teamName;
        this.teamImage = userTeamData.teamImage; // Assign the team image

        // Fetch season information
        await this.fetchSeasonData();

        // Fetch schedules for the current, previous, and next month
        await this.fetchSchedulesForMonth(this.currentMonth);
        await this.fetchSchedulesForMonth(
          format(subMonths(new Date(), 1), "yyyy-MM")
        );
        await this.fetchSchedulesForMonth(
          format(addMonths(new Date(), 1), "yyyy-MM")
        );
      } catch (error) {
        console.error("Error fetching initial data:", error);
      }
    },
    async fetchSeasonData() {
      try {
        const seasonResponse = await fetch(
          `https://api.powerplay.com/seasons/current`
        );
        const seasonData = await seasonResponse.json();
        /* The response should look like this:
          {
            "id": "67890",
            "leagues": [
              {
                "id": "league1",
                "teams": [
                  { "id": "12345", "name": "Team Awesome" },
                  // other teams...
                ]
              },
              // other leagues...
            ]
          }
        */
        this.seasonId = seasonData.id;

        // Assuming the season data includes leagues and teams
        const leagues = seasonData.leagues;
        leagues.forEach((league) => {
          if (league.teams.some((team) => team.id === this.teamId)) {
            this.leagueId = league.id;
          }
        });
      } catch (error) {
        console.error("Error fetching season data:", error);
      }
    },
    async fetchSchedulesForMonth(month) {
      try {
        const response = await fetch(
          `https://api.powerplay.com/teams/${this.teamId}/schedule?month=${month}`
        );
        const data = await response.json();
        /* The response should look like this:
          {
            "schedule": [
              {
                "start": "2024-06-01T10:00:00Z",
                "end": "2024-06-01T12:00:00Z",
                "event": "Game vs Team B",
                "description": "Home game at Stadium A",
                "score": "2-1"
              },
              // other games...
            ]
          }
        */

        this.schedule.push(...data.schedule);
        this.events.push(
          ...data.schedule.map((game) => ({
            start: game.start,
            end: game.end,
            title: game.event,
            description: game.description,
            score: game.score,
          }))
        );
      } catch (error) {
        console.error(`Error fetching schedule for month ${month}:`, error);
      }
    },
    async onRangeChange(range) {
      const startMonth = format(new Date(range.start), "yyyy-MM");
      const endMonth = format(new Date(range.end), "yyyy-MM");

      if (startMonth < this.currentMonth) {
        const previousMonth = format(
          subMonths(new Date(this.currentMonth), 1),
          "yyyy-MM"
        );
        await this.fetchSchedulesForMonth(previousMonth);
        this.currentMonth = previousMonth;
      }

      if (endMonth > this.currentMonth) {
        const nextMonth = format(
          addMonths(new Date(this.currentMonth), 1),
          "yyyy-MM"
        );
        await this.fetchSchedulesForMonth(nextMonth);
        this.currentMonth = nextMonth;
      }
    },
  },
};
</script>

<style scoped>
.team-schedule {
  box-sizing: 10;
}
.team-logo {
  width: 40px;
  height: 40px;
  margin-right: 10px;
}
</style>