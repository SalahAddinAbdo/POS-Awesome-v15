<template>
  <v-row justify="center">
    <v-dialog v-model="closingDialog" max-width="900px">
      <v-card>
        <v-card-title>
          <span class="text-h5 text-primary">{{
            __('Closing POS Shift')
          }}</span>
        </v-card-title>
        <v-card-text class="pa-0">
          <v-container>
            <v-row>
              <v-col cols="12" class="pa-1">
                <v-data-table :headers="headers" :items="dialog_data.payment_reconciliation" item-key="mode_of_payment"
                  class="elevation-1" :items-per-page="itemsPerPage" hide-default-footer>
				
				<!-- Start code modified by Salah -->				
				<template v-slot:item.closing_amount="{ item, index }">
					<v-text-field
					  v-model="item.closing_amount"
					  type="text"
					  :label="frappe._('Closing Amount')"
					  :prefix="currencySymbol(pos_profile.currency)"
					  hide-details
					  density="compact"
					  @focus="activePopupIndex = index; activeKeypadIdx = index; selectedPayment = item">
					</v-text-field>

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
						<v-col cols="4"><v-btn block color="success" @click="activePopupIndex = null">{{ __("OK") }}</v-btn></v-col>
					  </v-row>
					</v-card>
				  </v-menu>
				</template>
				<!-- End code modified by Salah -->
		
                  <template v-slot:item.difference="{ item }">
                    {{ currencySymbol(pos_profile.currency) }}
                    {{
                      (item.difference = formatCurrency(
                        item.expected_amount - item.closing_amount
                      ))
                    }}</template>
                  <template v-slot:item.opening_amount="{ item }">
                    {{ currencySymbol(pos_profile.currency) }}
                    {{ formatCurrency(item.opening_amount) }}</template>
                  <template v-slot:item.expected_amount="{ item }">
                    {{ currencySymbol(pos_profile.currency) }}
                    {{ formatCurrency(item.expected_amount) }}</template>
                </v-data-table>
              </v-col>
            </v-row>
          </v-container>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="error" theme="dark" @click="close_dialog">{{
            __('Close')
          }}</v-btn>
          <v-btn color="success" theme="dark" @click="submit_dialog">{{
            __('Submit')
          }}</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-row>
</template>

<script>

import format from '../../format';
export default {
  mixins: [format],
  data: () => ({
    closingDialog: false,
    itemsPerPage: 20,
    dialog_data: {},
    pos_profile: '',
    headers: [
      {
        title: __('Mode of Payment'),
        value: 'mode_of_payment',
        align: 'start',
        sortable: true,
      },
      {
        title: __('Opening Amount'),
        align: 'end',
        sortable: true,
        value: 'opening_amount',
      },
      {
        title: __('Closing Amount'),
        value: 'closing_amount',
        align: 'end',
        sortable: true,
      },
    ],
    max25chars: (v) => v.length <= 20 || 'Input too long!', // TODO : should validate as number
    pagination: {},
	activeKeypadIdx: null,       // New code added aby Salah
	activePopupIndex: null,      // New code added aby Salah
	selectedPayment: {},         // New code added aby Salah
	dummyMenuModel: true,  // New code added aby Salah
  }),
  watch: {},

  methods: {
    close_dialog() {
      this.closingDialog = false;
    },
    submit_dialog() {
      this.eventBus.emit('submit_closing_pos', this.dialog_data);
      this.closingDialog = false;
    },
	// Satrt New code by Salah
	appendNumber(n) {
	  const target = this.dialog_data.payment_reconciliation[this.activeKeypadIdx];
	  if (target) {
		const current = target.closing_amount?.toString() || '';
		target.closing_amount = parseFloat(current + n);
	  }
	},
	clearAmount() {
	  const target = this.dialog_data.payment_reconciliation[this.activeKeypadIdx];
	  if (target) target.closing_amount = 0;
	},
	// End New code by Salah
  },

  created: function () {
    this.eventBus.on('open_ClosingDialog', (data) => {
      this.closingDialog = true;
      this.dialog_data = data;
    });
    this.eventBus.on('register_pos_profile', (data) => {
      this.pos_profile = data.pos_profile;
      if (!this.pos_profile.hide_expected_amount) {
        this.headers.push({
          title: __('Expected Amount'),
          value: 'expected_amount',
          align: 'end',
          sortable: false,
        });
        this.headers.push({
          title: __('Difference'),
          value: 'difference',
          align: 'end',
          sortable: false,
        });
      }
    });
  },
};
</script>
