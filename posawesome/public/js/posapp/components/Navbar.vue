<template>
  <nav>
    <v-app-bar height="40" class="elevation-2">
      <v-app-bar-nav-icon @click.stop="drawer = !drawer" class="text-grey"></v-app-bar-nav-icon>
      <v-img src="/assets/posawesome/js/posapp/components/pos/pos.png" alt="POS Awesome" max-width="32" class="mr-2"
        color="primary"></v-img>

      <v-spacer></v-spacer>
      <v-btn style="cursor: unset" variant="text" color="primary">
        <span right>{{ pos_profile.name }}</span>
      </v-btn>
      <div class="text-center">
        <v-menu
          location="bottom end"
          origin="top right"
          offset-y
        >
          <template v-slot:activator="{ props }">
            <v-btn color="primary" theme="dark" variant="text" v-bind="props">{{__('Menu')}}</v-btn> <!-- Code Modified by Salah -->
          </template>
          <v-card class="mx-auto" max-width="300" tile>
            <v-list density="compact" v-model="menu_item" color="primary">

              <v-list-item @click="close_shift_dialog" v-if="!pos_profile.posa_hide_closing_shift && item == 0">
                <template v-slot:prepend>
                  <v-icon icon="mdi-content-save-move-outline"></v-icon>
                </template>

                <v-list-item-title>{{
                  __('Close Shift')
                }}</v-list-item-title>

              </v-list-item>
              <v-list-item @click="print_last_invoice" v-if="
                pos_profile.posa_allow_print_last_invoice &&
                this.last_invoice
              ">
                <template v-slot:prepend>
                  <v-icon icon="mdi-printer"></v-icon>
                </template>

                <v-list-item-title>{{
                  __('Print Last Invoice')
                }}</v-list-item-title>

              </v-list-item>
              <v-divider class="my-0"></v-divider>
              <v-list-item @click="logOut">
                <template v-slot:prepend>
                  <v-icon icon="mdi-logout"></v-icon>
                </template>

                <v-list-item-title>{{ __('Logout') }}</v-list-item-title>

              </v-list-item>
              <v-list-item @click="go_desk">
                <template v-slot:prepend>
                  <v-icon icon="mdi-information-outline"></v-icon>
                </template>

                <v-list-item-title>{{ __('Desktop') }}</v-list-item-title> <!-- Code Modified by Salah -->

              </v-list-item>

            </v-list>
          </v-card>
        </v-menu>
      </div>
    </v-app-bar>
    <v-navigation-drawer v-model="drawer" v-model:mini-variant="mini" class="bg-primary margen-top" width="60">
      <v-list theme="dark">
        <v-list-item class="px-2">
          <template v-slot:prepend>
            <v-avatar>
              <v-img :src="company_img"></v-img>
            </v-avatar>
          </template>

          <v-list-item-title>{{ company }}</v-list-item-title>

          <v-btn icon @click.stop="mini = !mini">
            <v-icon icon="mdi-chevron-left"></v-icon>
          </v-btn>
        </v-list-item>
        <!-- <MyPopup/> -->
        <v-list v-model="item" color="white">
          <v-list-item v-for="item in items" :key="item.text" @click="changePage(item.text)">
            <template v-slot:prepend>
              <v-icon :icon="item.icon"></v-icon>
            </template>

            <v-list-item-title>
              <div v-text="item.text"></div>
            </v-list-item-title>

          </v-list-item>
        </v-list>
      </v-list>
    </v-navigation-drawer>
    <v-snackbar v-model="snack" :timeout="5000" :color="snackColor" location="top">
      {{ snackText }}
    </v-snackbar>
    <v-dialog v-model="freeze" persistent max-width="290">
      <v-card>
        <v-card-title class="text-h5">
          {{ freezeTitle }}
        </v-card-title>
        <v-card-text>{{ freezeMsg }}</v-card-text>
      </v-card>
    </v-dialog>
  </nav>
</template>

<script>

export default {
  // components: {MyPopup},
  data() {
    return {
      drawer: false,
      mini: true,
      item: 0,
      items: [{ text: 'POS', icon: 'mdi-network-pos' }],
      page: '',
      fav: true,
      menu: false,
      message: false,
      hints: true,
      menu_item: 0,
      snack: false,
      snackColor: '',
      snackText: '',
      company: 'POS Awesome',
      company_img: '/assets/erpnext/images/erpnext-logo.svg',
      pos_profile: '',
      freeze: false,
      freezeTitle: '',
      freezeMsg: '',
      last_invoice: '',
      silent_print_socket: null,
    };
  },
  methods: {
    changePage(key) {
      this.$emit('changePage', key);
    },
    go_desk() {
      frappe.set_route('/');
      location.reload();
    },
    go_about() {
      const win = window.open(
        '/app',
        '_blank'
      ); // Code Modified by Salah
      win.focus();
    },
    close_shift_dialog() {
      this.eventBus.emit('open_closing_dialog');
    },
    show_message(data) {
      this.snack = true;
      this.snackColor = data.color;
      this.snackText = data.title;
    },
    logOut() {
      var me = this;
      me.logged_out = true;
      return frappe.call({
        method: 'logout',
        callback: function (r) {
          if (r.exc) {
            return;
          }
          frappe.set_route('/login');
          location.reload();
        },
      });
    },
    print_last_invoice() {
      if (!this.last_invoice) return;
      if(this.silent_print_socket){
        this.silent_print_invoice(this.last_invoice)

      } else {
        const print_format =
          this.pos_profile.print_format_for_online ||
          this.pos_profile.print_format;
        const letter_head = this.pos_profile.letter_head || 0;
        const url =
          frappe.urllib.get_base_url() +
          '/printview?doctype=Sales%20Invoice&name=' +
          this.last_invoice +
          '&trigger_print=1' +
          '&format=' +
          print_format +
          '&no_letterhead=' +
          letter_head;
        const printWindow = window.open(url, 'Print');
        printWindow.addEventListener(
          'load',
          function () {
            printWindow.print();
          },
          true
        );
      }
    },
    // pico silent print
    async get_user_default_printer() {
        try {
            const { message } = await frappe.call(
              "pico_silent_print.api.get_user_default_printer"
            );
            return message;
        } catch (e) {
            frappe.msgprint({
              title    : __("Silent Print"),
              indicator: "red",
              message  : __(`Cannot get default printer for this user: <b>${frappe.user.name}</b>`),
            });
            throw e;
        }
    },
    async silent_print_invoice(invoice_name){
      console.log("from: silent_print_invoice- ",invoice_name)
      const printer = await this.get_user_default_printer();
      if (!printer) return;

      const { message: { pdf_base64 } } = await frappe.call({
          method : "pico_silent_print.api.create_pdf",
          args : {
            doctype : "Sales Invoice",
            name : invoice_name,
            printer,
            no_letterhead: 1,
          },
      });

      this.send_to_printer(printer.printer, pdf_base64);

      // Extra item‑group slips for sales invoices
      // if (this.frm.doctype === "Sales Invoice") {
          const { message: itemPrints } = await frappe.call({
              method: "pico_silent_print.api.create_item_groups_pdfs",
              args  : { "invoice_name": invoice_name },
          });
          itemPrints.forEach(p =>
            this.send_to_printer(p.printer_name, p.pdf_base64)
          );
        // }
    },
    send_to_printer(printerId, pdfBase64){
      this.silent_print_socket.submit({
            type        : printerId,
            url         : "file.pdf",
            file_content: pdfBase64,
            id          : Date.now(),
        });
    }
  },
  created: function () {
    this.$nextTick(function () {

      // pico silent print
      // this.boot_silent_print_service()
      this.eventBus.on('pico_register_silent_print', (data) => {
        console.log("from pico-sp lestiner:", data)
        this.silent_print_socket = data
      });

      this.eventBus.on('show_message', (data) => {
        console.log("GOT Something: <s>")
        this.show_message(data);
      });
      this.eventBus.on('set_company', (data) => {
        this.company = data.name;
        this.company_img = data.company_logo
          ? data.company_logo
          : this.company_img;
      });
      this.eventBus.on('register_pos_profile', (data) => {
        this.pos_profile = data.pos_profile;
        const payments = { text: 'Payments', icon: 'mdi-cash-register' };
        if (
          this.pos_profile.posa_use_pos_awesome_payments &&
          this.items.length !== 2
        ) {
          this.items.push(payments);
        }
      });
      this.eventBus.on('set_last_invoice', (data) => {
        this.last_invoice = data;
      });
      this.eventBus.on('freeze', (data) => {
        this.freeze = true;
        this.freezeTitle = data.title;
        this.freezeMsg = data.msg;
      });
      this.eventBus.on('unfreeze', () => {
        this.freeze = false;
        this.freezTitle = '';
        this.freezeMsg = '';
      });

    });
  },
};
</script>

<style scoped>
.margen-top {
  margin-top: 0px;
}
</style>
