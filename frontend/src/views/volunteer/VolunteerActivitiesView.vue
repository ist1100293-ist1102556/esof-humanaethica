<template>
  <div>
    <v-card class="table">
      <v-data-table
        :headers="headers"
        :items="activities"
        :search="search"
        disable-pagination
        :hide-default-footer="true"
        :mobile-breakpoint="0"
        data-cy="volunteerActivitiesTable"
      >
        <template v-slot:top>
          <v-card-title>
            <v-text-field
              v-model="search"
              append-icon="search"
              label="Search"
              class="mx-2"
            />
            <v-spacer />
          </v-card-title>
        </template>
        <template v-slot:[`item.themes`]="{ item }">
          <v-chip v-for="theme in item.themes" v-bind:key="theme.id">
            {{ theme.completeName }}
          </v-chip>
        </template>
        <template v-slot:[`item.action`]="{ item }">
          <v-tooltip v-if="item.state === 'APPROVED'" bottom>
            <template v-slot:activator="{ on }">
              <v-icon
                class="mr-2 action-button"
                color="red"
                v-on="on"
                data-cy="reportButton"
                @click="reportActivity(item)"
                >warning</v-icon
              >
            </template>
            <span>Report Activity</span>
          </v-tooltip>
          <v-tooltip v-if="canEnroll(item)" bottom>
            <template v-slot:activator="{ on }">
              <v-icon
                class="mr-2 action-button"
                color="blue"
                v-on="on"
                @click="createEnrollment(item)"
                data-cy="applyButton"
                >fa-solid fa-right-to-bracket</v-icon
              >
            </template>
            <span>Apply for Activity</span>
          </v-tooltip>
          <v-tooltip v-if="canReview(item)" bottom>
            <template v-slot:activator="{ on }">
              <v-icon
                class="mr-2 action-button"
                color="blue"
                @click="createAssessment(item)"
                v-on="on"
                data-cy="writeAssessmentButton"
                >fa-solid fa-pen-to-square</v-icon
              >
            </template>
            <span>Write an assessment</span>
          </v-tooltip>
        </template>
      </v-data-table>
      <enrollment-dialog
        v-if="currentActivity && enrollmentDialog"
        v-model="enrollmentDialog"
        :activityId="currentActivity.id"
        v-on:save-enrollment="onSaveEnrollment"
        v-on:close-enrollment-dialog="onCloseEnrollmentDialog"
      />
      <assessment-dialog
        v-if="currentActivity && assessmentDialog"
        v-model="assessmentDialog"
        :institutionId="currentActivity.institution.id"
        v-on:save-assessment="onSaveAssessment"
        v-on:close-assessment-dialog="onCloseAssessmentDialog"
      />
    </v-card>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import RemoteServices from '@/services/RemoteServices';
import Activity from '@/models/activity/Activity';
import { show } from 'cli-cursor';
import EnrollmentDialog from '@/views/volunteer/EnrollmentDialog.vue';
import AssessmentDialog from '@/views/volunteer/AssessmentDialog.vue';
import Enrollment from '@/models/enrollment/Enrollment';
import Participation from '@/models/participation/Participation';
import Assessment from '@/models/assessment/Assessment';

@Component({
  methods: { show },
  components: {
    'enrollment-dialog': EnrollmentDialog,
    'assessment-dialog': AssessmentDialog,
  },
})
export default class VolunteerActivitiesView extends Vue {
  activities: Activity[] = [];
  search: string = '';

  enrollmentDialog: boolean = false;
  assessmentDialog: boolean = false;
  currentActivity: Activity | null = null;

  enrollments: Enrollment[] = [];

  participations: Participation[] = [];

  assessments: Assessment[] = [];

  headers: object = [
    {
      text: 'Name',
      value: 'name',
      align: 'left',
      width: '5%',
    },
    {
      text: 'Region',
      value: 'region',
      align: 'left',
      width: '5%',
    },
    {
      text: 'Participants',
      value: 'participantsNumberLimit',
      align: 'left',
      width: '5%',
    },
    {
      text: 'Themes',
      value: 'themes',
      align: 'left',
      width: '5%',
    },
    {
      text: 'Description',
      value: 'description',
      align: 'left',
      width: '30%',
    },
    {
      text: 'State',
      value: 'state',
      align: 'left',
      width: '5%',
    },
    {
      text: 'Start Date',
      value: 'formattedStartingDate',
      align: 'left',
      width: '5%',
    },
    {
      text: 'End Date',
      value: 'formattedEndingDate',
      align: 'left',
      width: '5%',
    },
    {
      text: 'Application Deadline',
      value: 'formattedApplicationDeadline',
      align: 'left',
      width: '5%',
    },
    {
      text: 'Actions',
      value: 'action',
      align: 'left',
      sortable: false,
      width: '5%',
    },
  ];

  async created() {
    await this.$store.dispatch('loading');
    try {
      this.activities = await RemoteServices.getActivities();
    } catch (error) {
      await this.$store.dispatch('error', error);
    }
    try {
      this.enrollments = await RemoteServices.getVolunteerEnrollment();
    } catch (error) {
      await this.$store.dispatch('error', error);
    }
    try {
      this.participations = await RemoteServices.getVolunteerParticipations();
    } catch (error) {
      await this.$store.dispatch('error', error);
    }
    try {
      this.assessments = await RemoteServices.getVolunteerAssessments();
    } catch (error) {
      await this.$store.dispatch('error', error);
    }
    await this.$store.dispatch('clearLoading');
  }

  async reportActivity(activity: Activity) {
    if (activity.id !== null) {
      try {
        const result = await RemoteServices.reportActivity(
          this.$store.getters.getUser.id,
          activity.id,
        );
        this.activities = this.activities.filter((a) => a.id !== activity.id);
        this.activities.unshift(result);
      } catch (error) {
        await this.$store.dispatch('error', error);
      }
    }
  }

  createEnrollment(activity: Activity) {
    this.currentActivity = activity;
    this.enrollmentDialog = true;
  }

  onSaveEnrollment(enrollment: Enrollment) {
    this.enrollments.unshift(enrollment);
    this.enrollmentDialog = false;
    this.currentActivity = null;
  }

  onCloseEnrollmentDialog() {
    this.enrollmentDialog = false;
    this.currentActivity = null;
  }

  canEnroll(activity: Activity | null) {
    return (
      activity &&
      new Date(activity.applicationDeadline) > new Date() &&
      !this.enrollments.some((e: Enrollment) => e.activityId === activity.id)
    );
  }

  createAssessment(activity: Activity) {
    this.currentActivity = activity;
    this.assessmentDialog = true;
  }

  onSaveAssessment(assessment: Assessment) {
    this.assessments.unshift(assessment);
    this.assessmentDialog = false;
    this.currentActivity = null;
  }

  onCloseAssessmentDialog() {
    this.assessmentDialog = false;
    this.currentActivity = null;
  }

  canReview(activity: Activity | null) {
    return (
      activity &&
      new Date(activity.endingDate) < new Date() &&
      !this.assessments.some(
        (a: Assessment) => a.institutionId === activity.institution.id,
      ) &&
      this.participations.some(
        (p: Participation) => p.activityId === activity.id,
      )
    );
  }
}
</script>

<style lang="scss" scoped></style>
