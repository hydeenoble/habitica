<template lang="pug">
// @TODO: Move this to a member directory
div
  remove-member-modal(:member-to-remove='memberToRemove', :group-id='this.groupId' @member-removed='memberRemoved')
  b-modal#members-modal(:title="$t('createGuild')", size='md', :hide-footer='true')
    .header-wrap(slot="modal-header")
      .row
        .col-6
          h1(v-once) {{$t('members')}}
        .col-6
          button(type="button" aria-label="Close" class="close", @click='close()')
            span(aria-hidden="true") ×
      .row
        .form-group.col-6
          input.form-control.search(type="text", :placeholder="$t('search')", v-model='searchTerm')
        .col-5.offset-1
          span.dropdown-label {{ $t('sortBy') }}
          b-dropdown(:text="$t('sort')", right=true)
            b-dropdown-item(v-for='sortOption in sortOptions', @click='sort(sortOption.value)', :key='sortOption.value') {{sortOption.text}}
    .row(v-if='invites.length > 0')
      .col-6.offset-3.nav
        .nav-item(@click='viewMembers()', :class="{active: selectedPage === 'members'}") {{ $t('members') }}
        .nav-item(@click='viewInvites()', :class="{active: selectedPage === 'invites'}") {{ $t('invites') }}
    div(v-if='selectedPage === "members"')
      .row(v-for='(member, index) in sortedMembers')
        .col-11.no-padding-left
          member-details(:member='member')
        .col-1.actions
          b-dropdown(right=true)
            .svg-icon.inline.dots(slot='button-content', v-html="icons.dots")
            b-dropdown-item(@click='removeMember(member, index)', v-if='isLeader')
              span.dropdown-icon-item
                .svg-icon.inline(v-html="icons.removeIcon", v-if='isLeader')
                span.text {{$t('removeMember')}}
            b-dropdown-item(@click='sendMessage(member)')
              span.dropdown-icon-item
                .svg-icon.inline(v-html="icons.messageIcon")
                span.text {{$t('sendMessage')}}
            b-dropdown-item(@click='sort(option.value)', v-if='isLeader')
              span.dropdown-icon-item
                .svg-icon.inline(v-html="icons.starIcon")
                span.text {{$t('promoteToLeader')}}
            b-dropdown-item(@click='addManager(member)', v-if='isLeader && groupIsSubscribed')
              span.dropdown-icon-item
                .svg-icon.inline(v-html="icons.starIcon")
                span.text {{$t('addManager')}}
            b-dropdown-item(@click='removeManager(member)', v-if='isLeader && groupIsSubscribed')
              span.dropdown-icon-item
                .svg-icon.inline(v-html="icons.removeIcon")
                span.text {{$t('removeManager2')}}
      .row(v-if='groupId === "challenge"')
        .col-12.text-center
          button.btn.btn-secondary(@click='loadMoreMembers()') {{ $t('loadMore') }}
      .row.gradient(v-if='members.length > 3')
    div(v-if='selectedPage === "invites"')
      .row(v-for='(member, index) in invites')
        .col-11.no-padding-left
          member-details(:member='member')
        .col-1.actions
          b-dropdown(right=true)
            .svg-icon.inline.dots(slot='button-content', v-html="icons.dots")
            b-dropdown-item(@click='removeInvite(member, index)', v-if='isLeader')
              span.dropdown-icon-item
                .svg-icon.inline(v-html="icons.removeIcon", v-if='isLeader')
                span.text {{$t('removeInvite')}}
    .modal-footer
      button.btn.btn-primary(@click='close()') {{ $t('close') }}
</template>

<style lang='scss'>
  #members-modal {
    .small-text, .character-name {
      color: #878190;
    }

    .no-padding-left, .modal-body {
      padding-left: 0;
    }

    .member-details {
      margin: 0;
    }

    .actions .dropdown-toggle::after {
      content: none !important;
    }
  }
</style>

<style lang='scss' scoped>
  header {
    background-color: #edecee;
    border-radius: 4px 4px 0 0;
  }

  .header-wrap {
    width: 100%;
  }

  h1 {
    color: #4f2a93;
  }

  .actions {
    padding-top: 5em;

    .dots {
      height: 16px;
      width: 4px;
    }

    .btn-group {
      margin-left: -2em;
      margin-top: -2em;
    }

    .action-icon {
      margin-right: 1em;
    }
  }

  #members-modal_modal_body {
    padding: 0;
    max-height: 450px;

    .col-8 {
      margin-left: 0;
    }

    .member-details {
      margin: 0;
    }

    .member-stats {
      width: 382px;
      height: 147px;
    }

    .gradient {
      background-image: linear-gradient(to bottom, rgba(255, 255, 255, 0), #ffffff);
      height: 50px;
      width: 100%;
      position: absolute;
      bottom: 0px;
      margin-left: -15px;
    }
  }

  .dropdown-icon-item .svg-icon {
    width: 20px;
  }

  .nav {
    font-weight: bold;
    margin-bottom: .5em;
    margin-top: .5em;
  }

  .nav-item {
    display: inline-block;
    font-size: 16px;
    margin: 0 auto;
    padding: .5em;
    color: #878190;
  }

  .nav-item:hover, .nav-item.active {
    color: #4f2a93;
    border-bottom: 2px solid #4f2a93;
    cursor: pointer;
  }
</style>

<script>
import sortBy from 'lodash/sortBy';
import bModal from 'bootstrap-vue/lib/components/modal';
import bDropdown from 'bootstrap-vue/lib/components/dropdown';
import bDropdownItem from 'bootstrap-vue/lib/components/dropdown-item';
import { mapState } from 'client/libs/store';

import removeMemberModal from 'client/components/members/removeMemberModal';
import MemberDetails from '../memberDetails';
import removeIcon from 'assets/members/remove.svg';
import messageIcon from 'assets/members/message.svg';
import starIcon from 'assets/members/star.svg';
import dots from 'assets/svg/dots.svg';

export default {
  props: ['hideBadge'],
  components: {
    bModal,
    bDropdown,
    bDropdownItem,
    MemberDetails,
    removeMemberModal,
  },
  data () {
    return {
      sortOption: '',
      selectedPage: 'members',
      members: [],
      invites: [],
      memberToRemove: {},
      sortOptions: [
        {
          value: 'level',
          text: this.$t('tier'),
        },
        {
          value: 'name',
          text: this.$t('name'),
        },
        {
          value: 'lvl',
          text: this.$t('level'),
        },
        {
          value: 'class',
          text: this.$t('class'),
        },
      ],
      searchTerm: '',
      icons: Object.freeze({
        removeIcon,
        messageIcon,
        starIcon,
        dots,
      }),
      userIdToMessage: '',
    };
  },
  mounted () {
    this.getMembers();
  },
  computed: {
    ...mapState({user: 'user.data'}),
    isLeader () {
      if (!this.group || !this.group.leader) return false;
      return this.user._id === this.group.leader || this.user._id === this.group.leader._id;
    },
    groupIsSubscribed () {
      return this.group.purchased.active;
    },
    group () {
      return this.$store.state.memberModalOptions.group;
    },
    groupId () {
      return this.$store.state.memberModalOptions.groupId || this.group._id;
    },
    challengeId () {
      return this.$store.state.memberModalOptions.challengeId;
    },
    sortedMembers () {
      let sortedMembers = this.members;

      if (this.searchTerm) {
        sortedMembers = sortedMembers.filter(member => {
          return member.profile.name.toLowerCase().indexOf(this.searchTerm.toLowerCase) !== -1;
        });
      }

      if (!this.sortOption) return sortedMembers;

      sortedMembers = sortBy(this.members, [(member) => {
        if (this.sortOption === 'tier') {
          if (!member.contributor) return;
          return member.contributor.level;
        } else if (this.sortOption === 'name') {
          return member.profile.name;
        } else if (this.sortOption === 'lvl') {
          return member.stats.lvl;
        } else if (this.sortOption === 'class') {
          return member.stats.class;
        }
      }]);

      return sortedMembers;
    },
  },
  watch: {
    groupId () {
      // @TOOD: We might not need this since groupId is computed now
      this.getMembers();
    },
    group () {
      this.getMembers();
    },
  },
  methods: {
    sendMessage (member) {
      this.$root.$emit('habitica::new-inbox-message', {
        userIdToMessage: member._id,
        userName: member.profile.name,
      });
    },
    async getMembers () {
      let groupId = this.groupId;
      if (groupId && groupId !== 'challenge') {
        let members = await this.$store.dispatch('members:getGroupMembers', {
          groupId,
          includeAllPublicFields: true,
        });
        this.members = members;

        let invites = await this.$store.dispatch('members:getGroupInvites', {
          groupId,
          includeAllPublicFields: true,
        });
        this.invites = invites;
      }

      if (this.$store.state.memberModalOptions.viewingMembers.length > 0) {
        this.members = this.$store.state.memberModalOptions.viewingMembers;
      }
    },
    async clickMember (uid, forceShow) {
      let user = this.$store.state.user.data;

      if (user._id === uid && !forceShow) {
        if (this.$route.name === 'tasks') {
          this.$route.router.go('options.profile.avatar');
          return;
        }

        this.$route.router.go('tasks');
        return;
      }

      await this.$store.dispatch('members:selectMember', {
        memberId: uid,
      });

      this.$root.$emit('show::modal', 'members-modal');
    },
    async removeMember (member, index) {
      this.memberToRemove = member;
      this.memberToRemove.index = index;
      this.$root.$emit('show::modal', 'remove-member');
    },
    memberRemoved () {
      this.members.splice(this.memberToRemove.index, 1);
      this.group.memberCount -= 1;
      this.memberToRemove =  {};
    },
    async addManager (memberId) {
      await this.$store.dispatch('guilds:addManager', {
        groupId: this.groupId,
        memberId,
      });
      alert(this.$t('managerAdded'));
    },
    async removeManager (memberId) {
      await this.$store.dispatch('guilds:removeManager', {
        groupId: this.groupId,
        memberId,
      });
      alert(this.$t('managerRemoved'));
    },
    close () {
      this.$root.$emit('hide::modal', 'members-modal');
    },
    sort (option) {
      this.sortOption = option;
    },
    async loadMoreMembers () {
      const lastMember = this.members[this.members.length - 1];
      if (!lastMember) return;

      let newMembers = await this.$store.dispatch('members:getChallengeMembers', {
        challengeId: this.challengeId,
        lastMemberId: lastMember._id,
      });

      this.members = this.members.concat(newMembers);
    },
    viewMembers () {
      this.selectedPage = 'members';
    },
    viewInvites () {
      this.selectedPage = 'invites';
    },
    async removeInvite (member, index) {
      this.invites.splice(index, 1);
      await this.$store.dispatch('members:removeMember', {
        memberId: member._id,
        groupId: this.groupId,
      });
      this.viewMembers();
    },
  },
};
</script>
