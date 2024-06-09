<template>
  <div>
    <v-card
      class="selection mx-auto grey lighten-5"
      style="max-height: 80vh; height: 80vh"
    >
      <v-card-title>
        <v-row no-gutters align="center" justify="center">
          <v-col cols="6">
            <span class="text-h6 primary--text">{{ __('Coupons') }}</span>
          </v-col>
          <v-col cols="4">
            <v-text-field
              dense
              outlined
              color="primary"
              :label="frappe._('Coupon')"
              background-color="white"
              hide-details
              v-model="new_coupon"
              class="mr-4"
            ></v-text-field>
          </v-col>
          <v-col cols="2">
            <v-btn
              class="pa-1"
              color="success"
              dark
              @click="add_coupon(new_coupon)"
              >{{ __('add') }}</v-btn
            >
          </v-col>
        </v-row>
      </v-card-title>
      <div class="my-0 py-0 overflow-y-auto" style="max-height: 75vh">
        <template @mouseover="style = 'cursor: pointer'">
          <v-data-table
            :headers="items_headers"
            :items="posa_coupons"
            :single-expand="singleExpand"
            :expanded.sync="expanded"
            item-key="coupon"
            class="elevation-1"
            :items-per-page="itemsPerPage"
            hide-default-footer
          >
            <template v-slot:item.applied="{ item }">
              <v-simple-checkbox
                v-model="item.applied"
                disabled
              ></v-simple-checkbox>
            </template>
          </v-data-table>
        </template>
      </div>
    </v-card>

    <v-card
      flat
      style="max-height: 11vh; height: 11vh"
      class="cards mb-0 mt-3 py-0"
    >
      <v-row align="start" no-gutters>
        <v-col cols="12">
          <v-btn
            block
            class="pa-1"
            large
            color="warning"
            dark
            @click="back_to_invoice"
            >{{ __('Back') }}</v-btn
          >
        </v-col>
      </v-row>
    </v-card>
  </div>
</template>

<script>
import { evntBus } from '../../bus';
export default {
  data: () => ({
    loading: false,
    pos_profile: '',
    customer: '',
    posa_coupons: [],
    new_coupon: null,
    itemsPerPage: 1000,
    singleExpand: true,
    membershipcard: null,
    items_headers: [
      { text: __('Coupon'), value: 'coupon_code', align: 'start' },
      { text: __('Type'), value: 'type', align: 'start' },
      { text: __('Offer'), value: 'pos_offer', align: 'start' },
      { text: __('Applied'), value: 'applied', align: 'start' },
    ],
  }),

  computed: {
    couponsCount() {
      return this.posa_coupons.length;
    },
    appliedCouponsCount() {
      return this.posa_coupons.filter((el) => !!el.applied).length;
    },
  },

  methods: {
    back_to_invoice() {
      evntBus.$emit('show_coupons', 'false');
    },

    add_coupon2(new_coupon) {
      if (!this.customer || !new_coupon) return;
      const exist = this.posa_coupons2.find(el => el.name == new_coupon);
      if (exist) {
        evntBus.$emit('show_mesage', {
          text: __('Member Is Added !'),
          color: 'success',
        });
        return;
      }
      const vm = this;
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.get_membership_card_add',
        args: {
          customer: vm.customer,
        },
        callback: function (r) {
          if (r.message) {
            const cards = r.message;
            if (cards.length === 0) {
              evntBus.$emit('show_mesage', {
                text: __('No valid membership card found.'),
                color: 'error',
              });
              return;
            }

            cards.forEach(card => {
              vm.posa_coupons2.push({
                coupon2: card.name,
                name: card.name,
                max_use: card.max_use,
                applied: 0,
                pos_offer: card.pos_offer,
                customer: vm.customer,
              });
            });

            evntBus.$emit('show_mesage', {
              text: __('Membership card(s) added successfully!'),
              color: 'success',
            });

            vm.new_coupon = null;
          }
        },
      });
    },
    setActiveGiftCoupons() {
      if (!this.customer) return;
      const vm = this;
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.get_active_gift_coupons',
        args: {
          customer: vm.customer,
          company: vm.pos_profile.company,
          
        },
        callback: function (r) {
          if (r.message) {
            const coupons = r.message;
            coupons.forEach((coupon_code) => {
              vm.add_coupon(coupon_code);
            });
          }
        },
      });
    },

    updatePosCoupons(offers) {
      this.posa_coupons.forEach((coupon) => {
        const offer = offers.find(
          (el) => el.offer_applied && el.coupon == coupon.coupon
        );
        if (offer) {
          coupon.applied = 1;
        } else {
          coupon.applied = 0;
        }
      });
    },

    removeCoupon(reomove_list) {
      this.posa_coupons = this.posa_coupons.filter(
        (coupon) => !reomove_list.includes(coupon.coupon)
      );
    },
    updateInvoice() {
      evntBus.$emit('update_invoice_coupons', this.posa_coupons);
    },
    updateCounters() {
      evntBus.$emit('update_coupons_counters', {
        couponsCount: this.couponsCount,
        appliedCouponsCount: this.appliedCouponsCount,
      });
    },
  },

  watch: {
    posa_coupons: {
      deep: true,
      handler() {
        this.updateInvoice();
        this.updateCounters();
      },
    },
  },

  created: function () {
    evntBus.$on('update_membershipcard', (membershipcard) => {
      if (this.membershipcard !== membershipcard) {
        this.membershipcard = membershipcard;
        this.setActiveGiftCoupons(); // Call the necessary method to update coupons
      }
    });
    this.$nextTick(function () {
      evntBus.$on('register_pos_profile', (data) => {
        this.pos_profile = data.pos_profile;
      });
    });
   // Assuming this code is within a Vue component or similar structure

// Event listener for updating customer
evntBus.$on('update_customer', (customer) => {
  if (this.customer != customer) {
    const to_remove = [];
    this.posa_coupons.forEach((el) => {
      if (el.type == 'Promotional') {
        el.customer = customer;
      } else {
        to_remove.push(el.coupon);
      }
    });
    this.customer = customer;
    if (to_remove.length) {
      this.removeCoupon(to_remove);
    }   
  }
  this.setActiveGiftCoupons();
});

evntBus.$on('show_member', (membershipcard) => {
  // Assuming membershipcard is the object you want to update
  // Update the membershipcard data accordingly
  if (this.membershipcard !== membershipcard) {
    this.membershipcard = membershipcard;
    this.setActiveGiftCoupons(); // Call the necessary method to update coupons
  }
});

  },
};
</script>
