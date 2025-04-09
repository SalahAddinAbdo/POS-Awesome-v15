<template>
  <v-row justify="center">
    <v-dialog v-model="isOpen" persistent max-width="600px">
      <!-- <template v-slot:activator="{ on, attrs }">
        <v-btn color="primary" theme="dark" v-bind="attrs" v-on="on">Open Dialog</v-btn>
      </template>-->
      <v-card>
        <v-card-title>
          <span class="text-h5 text-primary">{{
            __('Create POS Opening Shift')
          }}</span>
        </v-card-title>
        <v-card-text>
          <v-container>
            <v-row>
              <v-col cols="12">
                <v-autocomplete :items="companies" :label="frappe._('Company')" v-model="company"
                  required></v-autocomplete>
              </v-col>
              <v-col cols="12">
                <v-autocomplete :items="pos_profiles" :label="frappe._('POS Profile')" v-model="pos_profile"
                  required></v-autocomplete>
              </v-col>
              <v-col cols="12">
                <v-data-table :headers="payments_methods_headers" :items="payments_methods" item-key="mode_of_payment"
                  class="elevation-1" :items-per-page="itemsPerPage" hide-default-footer>
					<!-- Start code modified by Salah -->	
					<template v-slot:item.amount="{ item, index }">
					  <v-text-field
						v-model="item.amount"
						type="text"
						:label="frappe._('Opening Amount')"
						:prefix="currencySymbol(item.currency)"
						hide-details
						density="compact"
						@focus="activePopupIndex = index; activeKeypadIdx = index; selectedPayment = item; editingField = 'amount'"
					  ></v-text-field>

					  <v-menu
						v-if="activePopupIndex === index"
						v-model="dummyMenuModel"
						activator="parent"
						:close-on-content-click="false"
						@click:outside="activePopupIndex = null"
						offset-y
					  >
						<v-card class="pa-2 popup-numbers" style="min-width: 300px; max-width: 50%;" @click.stop>
						  <v-row dense>
							<v-col cols="4" v-for="n in 9" :key="n">
							  <v-btn block @click="appendNumber(n)">{{ n }}</v-btn>
							</v-col>
							<v-col cols="4"><v-btn block @click="clearAmount()">C</v-btn></v-col>
							<v-col cols="4"><v-btn block @click="appendNumber(0)">0</v-btn></v-col>
							<v-col cols="4"><v-btn block color="success" @click="activePopupIndex = null">{{__("OK")}}</v-btn></v-col>
						  </v-row>
						</v-card>
					  </v-menu>
					</template>
					<!-- End code modified by Salah -->
                </v-data-table>
              </v-col>
            </v-row>
          </v-container>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <!-- Code modified by Salah -->
          <v-btn color="error" theme="dark" @click="go_desk">
            {{ __('Cancel') }}
          </v-btn>
          
          <!-- Code modified by Salah -->
          <v-btn color="success" :disabled="is_loading" theme="dark" @click="submit_dialog">
            {{ __('Submit') }}
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-row>
</template>

<script>

import format from '../../format';
export default {
  mixins: [format],
  props: ['dialog'],
  data() {
    return {
      isOpen: this.dialog ? this.dialog : false,
      dialog_data: {},
      is_loading: false,
      companies: [],
      company: '',
      pos_profiles_data: [],
      pos_profiles: [],
      pos_profile: '',
      payments_method_data: [],
      payments_methods: [],
      payments_methods_headers: [
        {
          title: __('Mode of Payment'),
          align: 'start',
          sortable: false,
          value: 'mode_of_payment',
        },
        {
          title: __('Opening Amount'),
          value: 'amount',
          align: 'center',
          sortable: false,
        },
      ],
      itemsPerPage: 100,
      max25chars: (v) => v.length <= 12 || 'Input too long!', // TODO : should validate as number
      pagination: {},
      snack: false, // TODO : need to remove
      snackColor: '', // TODO : need to remove
      snackText: '', // TODO : need to remove
		activePopupIndex: null, // New code added aby Salah
		activeKeypadIdx: null, // New code added aby Salah
		selectedPayment: {}, // New code added aby Salah
		dummyMenuModel: true, // New code added aby Salah
		editingField: '', // New code added aby Salah
    };
  },
  watch: {
    company(val) {
      this.pos_profiles = [];
      this.pos_profiles_data.forEach((element) => {
        if (element.company === val) {
          this.pos_profiles.push(element.name);
        }
        if (this.pos_profiles.length) {
          this.pos_profile = this.pos_profiles[0];
        } else {
          this.pos_profile = '';
        }
      });
    },
    pos_profile(val) {
      this.payments_methods = [];
      this.payments_method_data.forEach((element) => {
        if (element.parent === val) {
          this.payments_methods.push({
            mode_of_payment: element.mode_of_payment,
            amount: 0,
            currency: element.currency,
          });
        }
      });
    },
  },
  methods: {
    close_opening_dialog() {
      this.eventBus.emit('close_opening_dialog');
    },
    get_opening_dialog_data() {
      const vm = this;
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.get_opening_dialog_data',
        args: {},
        callback: function (r) {
          if (r.message) {
            r.message.companies.forEach((element) => {
              vm.companies.push(element.name);
            });
            vm.company = vm.companies[0];
            vm.pos_profiles_data = r.message.pos_profiles_data;
            vm.payments_method_data = r.message.payments_method;
          }
        },
      });
    },
    submit_dialog() {
      if (!this.payments_methods.length || !this.company || !this.pos_profile) {
        return;
      }
      this.is_loading = true;
      var vm = this;
      return frappe
        .call('posawesome.posawesome.api.posapp.create_opening_voucher', {
          pos_profile: this.pos_profile,
          company: this.company,
          balance_details: this.payments_methods,
        })
        .then((r) => {
          if (r.message) {
            vm.eventBus.emit('register_pos_data', r.message);
            vm.eventBus.emit('set_company', r.message.company);
            vm.close_opening_dialog();
            is_loading = false;
          }
        });
    },
    go_desk() {
      frappe.set_route('/');
      location.reload();
    },
	// Satrt New code by Salah										
	appendNumber(n) {
	  const target = this.payments_methods[this.activeKeypadIdx];
	  if (target) {
		const field = this.editingField || 'amount';
		const current = target[field]?.toString() || '';
		target[field] = parseFloat(current + n);
	  }
	},
	clearAmount() {
	  const target = this.payments_methods[this.activeKeypadIdx];
	  if (target) {
		const field = this.editingField || 'amount';
		target[field] = 0;
	  }
	},
	// End New code by Salah
  },
  created: function () {
    this.$nextTick(function () {
      this.get_opening_dialog_data();
    });
  },
};
</script>
