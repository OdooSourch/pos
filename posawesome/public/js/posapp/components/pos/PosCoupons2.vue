<template>
  <div>
    <v-card
      class="selection mx-auto grey lighten-5"
      style="max-height: 80vh; height: 80vh"
    >
      <v-card-title>
        <v-row no-gutters align="center" justify="center">
          <v-col cols="6">
            <span class="text-h6 primary--text">{{ __('Member') }}</span>
          </v-col>
          <v-col cols="4">
            <v-text-field
              dense
              outlined
              color="primary"
              :label="frappe._('Member')"
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
              @click="add_coupon2(new_coupon)"
              >{{ __('add') }}</v-btn
            >
          </v-col>
        </v-row>
      </v-card-title>
      <div class="my-0 py-0 overflow-y-auto" style="max-height: 75vh">
        <template @mouseover="style = 'cursor: pointer'">
          <v-data-table
            :headers="items_headers"
            :items="posa_coupons2"
            :single-expand="singleExpand"
            :expanded.sync="expanded"
            item-key="coupon2"
            class="elevation-1"
            :items-per-page="itemsPerPage"
            hide-default-footer
          >
            <template v-slot:item.applied="{ item }">
              <v-simple-checkbox
                v-model="item.applied"
                enabled
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
    membership : '',
    posa_coupons2: [],
    new_coupon: null,
    itemsPerPage: 1000,
    singleExpand: true,
    items_headers: [
      { text: __('Name'), value: 'name', align: 'start' },
      { text: __('Max Use'), value: 'max_use', align: 'start' },
      { text: __('Offer'), value: 'pos_offer', align: 'start' },
      { text: __('Applied'), value: 'applied', align: 'start' },
      { text: __('Type'), value: 'discount_type', align: 'start' },
      { text: __('Value'), value: 'discount_value', align: 'start' },
    ],
    discounts: [],
  }),
  computed: {
    couponsCount2() {
      return this.posa_coupons2.length;
    },
    appliedCouponsCount2() {
      return this.posa_coupons2.filter((el) => !!el.applied).length;
    },
  },
  methods: {
    back_to_invoice() {
      evntBus.$emit('show_coupons2', 'false');
      console.log(this.invoice_doc);

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
                        discount_type: card.discount_type,
                        discount_value: card.discount_value, // Added discount_value
                        customer: vm.customer,
                    });
                    vm.discounts = [];
                    if (card.discount_type === 'Discount Amount') {
                        vm.discounts.push({
                            type: 'amount',
                            value: card.discount_value
                        });
                    } else if (card.discount_type === 'Discount Percentage') {
                        vm.discounts.push({
                            type: 'percent',
                            value: card.discount_value
                        });
                    }
                });

                evntBus.$emit('show_mesage', {
                    text: __('Membership card(s) added successfully!'),
                    color: 'success',
                  });
                  vm.new_coupon = null;
                  evntBus.$emit('update_pos_offers', vm.posa_coupons2);
                }
              },
            });
          },
     applyDiscountToItems(card, apply) {
        this.allItems.forEach(item => {
            if (item.name === card.pos_offer) {
                if (apply) {
                    if (card.discount_type === 'Discount Amount') {
                        item.discount_amount = card.discount_value;
                    } else if (card.discount_type === 'Discount Percentage') {
                        item.discount_percentage = card.discount_value;
                    }
                } else {
                    item.discount_amount = 0;
                    item.discount_percentage = 0;
                }
            }
        });
        this.updateInvoice();
    },
    updateInvoice() {
        evntBus.$emit('update_invoice_coupons', this.posa_coupons2);
        evntBus.$emit('update_invoice_items', this.allItems);
    },
    setActiveGiftCoupons() {
      if (!this.customer) return;
      const vm = this;
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.get_active_gift_member',
        args: {
          customer: vm.customer,
          company: vm.pos_profile.company,  
        },
        callback: function (r) {
          if (r.message) {
            vm.membership_cards = r.message;
            vm.posa_coupons2 = r.message;
            vm.membership_cards = r.message.filter(item => item.name === vm.membershipcard);
            vm.posa_coupons2 = r.message.filter(item => item.name === vm.membershipcard);
          }
        },
      });
    },
    updatePosCoupons(offers) {
      this.posa_coupons2.forEach((coupon2) => {
        const offer = offers.find(
          (el) => el.offer_applied && el.coupon2 == coupon2.coupon2
        );
        if (offer) {
          coupon2.applied = 1;
        } else {
          coupon2.applied = 0;
        }
      });
    },
    removeCoupon(reomove_list) {
      this.posa_coupons2= this.posa_coupons2.filter(
        (coupon2) => !reomove_list.includes(coupon2.name)
      );
    },
    updateInvoice() {
      evntBus.$emit('update_invoice_coupons', this.posa_coupons2);
    },
    updateCounters() {
      evntBus.$emit('update_coupons_counters2', {
        couponsCount2: this.couponsCount2,
        appliedCouponsCount2: this.appliedCouponsCount2,
      });
    },
  },

  watch: {
    posa_coupons2: {
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
        this.setActiveGiftCoupons();
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
    this.posa_coupons2.forEach((el) => {
      if (el.type == 'Promotional') {
        el.customer = customer;
      } else {
        to_remove.push(el.coupon2);
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
