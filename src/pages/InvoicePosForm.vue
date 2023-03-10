<template>
    <PosContainer>        
          <!-- Page Header (Title, Buttons, etc) -->
          <template #header v-if="doc">      
            <StatusBadge :status="status" />
            <ExchangeRate
              v-if="doc.isMultiCurrency"
              :disabled="doc?.isSubmitted || doc?.isCancelled"
              :from-currency="fromCurrency"
              :to-currency="toCurrency"
              :exchange-rate="doc.exchangeRate"
              @change="
                async (exchangeRate) => await doc.set('exchangeRate', exchangeRate)
              "
            />
            <Button
              v-if="!doc.isCancelled && !doc.dirty"
              :icon="true"
              @click="routeTo(`/print/${doc.schemaName}/${doc.name}`)"
            >
              {{ t`Print` }}
            </Button>
            <Button
              :icon="true"
              v-if="!doc?.isSubmitted && doc.enableDiscounting"
              @click="toggleInvoiceSettings"
            >
              <feather-icon name="settings" class="w-4 h-4" />
            </Button>
            <DropdownWithActions :actions="actions()" />
            <Button
              v-if="doc?.notInserted || doc?.dirty"
              type="primary"
              @click="sync"
            >
              {{ t`Save` }}
            </Button>
            <Button
              v-if="!doc?.dirty && !doc?.notInserted && !doc?.submitted"
              type="primary"
              @click="submit"
              >{{ t`Submit` }}</Button
            >
          </template>
          <template #items v-if="doc">       
            <div>
              <div class="container x-auto" >
                <div :style="{display:'flex', justifyContent:'space-around', alignItems:'center'}">
                  <input
                    ref="input"
                    type="search"
                    autocomplete="off"
                    spellcheck="false"
                    :placeholder="t`Type to search...`"
                    v-model="inputValue"
                    @focus="search"
                    @input="getSearchInput"
                    class="
                      bg-gray-100
                      text-xl
                      focus:outline-none
                      w-50
                      placeholder-gray-500
                      text-gray-700
                      rounded-xl
                      p-1
                      mt-4
                      mb-3
                    "
                  />
                  <div>
                      <!-- barcode -->
                      <Barcode
                        @item-selected="(name) => doc.addItem(name)"
                      />   
                  </div>
                </div>

                <div class="flex flex-wrap">   
                  <!-- Column --> 
                  <div v-for="item in filteredItems" :key="item.name" class="my-1 px-1 w-full md:w-1/2 lg:my-4 lg:px-4 lg:w-1/3">                
                    <article class="overflow-hidden border rounded content-center h-60 pt-2 pl-2 pr-2">
                      <h3 class="text-base font-medium mr-1 truncate ...">{{ item.name }}</h3>    
    
                      <a href="#"  @click="addItem(item.name)">
                            <img  :alt="item.name" class="object-cover self-center h-40 w-120" :src="item.image" v-if="item.image">
                            <div  alt="no_image" class="object-cover self-center h-40 w-120 pt-10 " :src="item.image" v-else>
                              <feather-icon name="camera-off" class="w-4 h-4 m-auto" />
                            </div>
                      </a>
                      
                      <Button
                          class="font-semibold pl-5 pr-3 border border-gray-400 rounded shadow w-full mt-2"
                          :icon="true"
                          type="primary"
                          @click="addItem(item.name)"
                          :padding="false"                                   
                        >
                          <span class="mr-1">{{ `${fyo.format(item.rate, 'Currency')}` }}</span>                                  
                          <!-- <feather-icon name="plus" class="w-4 h-4" /> -->
                        </Button>                                                                                    
                    </article>                
                  </div>
                  </div>
              </div>
            </div>
            <!-- Pagination Footer -->
            <div class="mt-auto" v-if="filteredItems?.length">
              <hr />
              <Paginator
                :item-count="filteredItems?.length"
                @index-change="setPageIndices"
                class="px-4"
              />
            </div>
          </template>

          <!-- Invoice Form -->
          <template #body v-if="doc">
            <div>
              <!-- Invoice Form Data Entry -->
              <div class="m-4 grid grid-cols-3 gap-4 attempt-form-data-entry">
                <FormControl
                  input-class="font-semibold"
                  :border="true"
                  :df="getField('party')"
                  :value="doc.party"
                  @change="(value) => doc.set('party', value, true)"
                  @new-doc="(party) => doc.set('party', party.name, true)"
                  :read-only="doc?.submitted"
                />            
                <FormControl
                  :border="true"
                  :df="getField('account')"
                  :value="doc.account"
                  @change="(value) => doc.set('account', value)"
                  :read-only="doc?.submitted"
                />
                
              </div>
              <hr />
  
              <!-- Invoice Items Table -->
              <Table
                class="text-base"
                :df="getField('items')"
                :value="doc.items"
                :showHeader="true"
                :max-rows-before-overflow="4"
                @change="(value) => doc.set('items', value)"
                @editrow="toggleQuickEditDoc"
                :read-only="doc?.submitted"
              />
            </div>
            
            <!-- Invoice Form Footer -->
  
            <div v-if="doc.items?.length ?? 0" class="mt-auto">
              <hr />
              <div class="flex justify-between text-base m-4 gap-12">
                <div class="w-1/2 flex flex-col justify-between">
                  <!-- Discount Note -->
                  <p v-if="discountNote?.length" class="text-gray-600 text-sm">
                    {{ discountNote }}
                  </p>
                  <!-- Form Terms-->
                  <FormControl
                    :border="true"
                    v-if="!doc?.submitted || doc.terms"
                    :df="getField('terms')"
                    :value="doc.terms"
                    class="mt-auto"
                    @change="(value) => doc.set('terms', value)"
                    :read-only="doc?.submitted"
                  />
                </div>
  
                <!-- Totals -->
                <div class="w-1/2 gap-2 flex flex-col self-end ml-auto">
                  <!-- Subtotal -->
                  <div class="flex justify-between">
                    <div>{{ t`Subtotal` }}</div>
                    <div>{{ formattedValue('netTotal') }}</div>
                  </div>
                  <hr />
  
                  <!-- Discount Applied Before Taxes -->
                  <div
                    v-if="totalDiscount.float > 0 && !doc.discountAfterTax"
                    class="flex flex-col gap-2"
                  >
                    <div
                      class="flex justify-between"
                      v-if="itemDiscountAmount.float > 0"
                    >
                      <div>{{ t`Discount` }}</div>
                      <div>
                        {{ `- ${fyo.format(itemDiscountAmount, 'Currency')}` }}
                      </div>
                    </div>
                    <div class="flex justify-between" v-if="discountAmount.float > 0">
                      <div>{{ t`Invoice Discount` }}</div>
                      <div>{{ `- ${fyo.format(discountAmount, 'Currency')}` }}</div>
                    </div>
                    <hr v-if="doc.taxes?.length" />
                  </div>
  
                  <!-- Taxes -->
                  <div
                    v-if="doc.taxes?.length"
                    class="flex flex-col gap-2 max-h-12 overflow-y-auto"
                  >
                    <div
                      class="flex justify-between"
                      v-for="tax in doc.taxes"
                      :key="tax.name"
                    >
                      <div>{{ tax.account }}</div>
                      <div>
                        {{
                          fyo.format(
                            tax.amount,
                            {
                              fieldtype: 'Currency',
                              fieldname: 'amount',
                            },
                            tax
                          )
                        }}
                      </div>
                    </div>
                  </div>
                  <hr v-if="doc.taxes?.length" />
  
                  <!-- Discount Applied After Taxes -->
                  <div
                    v-if="totalDiscount.float > 0 && doc.discountAfterTax"
                    class="flex flex-col gap-2"
                  >
                    <div
                      class="flex justify-between"
                      v-if="itemDiscountAmount.float > 0"
                    >
                      <div>{{ t`Discount` }}</div>
                      <div>
                        {{ `- ${fyo.format(itemDiscountAmount, 'Currency')}` }}
                      </div>
                    </div>
                    <div class="flex justify-between" v-if="discountAmount.float > 0">
                      <div>{{ t`Invoice Discount` }}</div>
                      <div>{{ `- ${fyo.format(discountAmount, 'Currency')}` }}</div>
                    </div>
                    <hr />
                  </div>
  
                  <!-- Grand Total -->
                  <div
                    class="
                      flex
                      justify-between
                      text-green-600
                      font-semibold
                      text-base
                    "
                  >
                    <div>{{ t`Grand Total` }}</div>
                    <div>{{ formattedValue('grandTotal') }}</div>
                  </div>
  
                  <!-- Base Grand Total -->
                  <div
                    v-if="doc.isMultiCurrency"
                    class="
                      flex
                      justify-between
                      text-green-600
                      font-semibold
                      text-base
                    "
                  >
                    <div>{{ t`Base Grand Total` }}</div>
                    <div>{{ formattedValue('baseGrandTotal') }}</div>
                  </div>
  
                  <!-- Outstanding Amount -->
                  <hr v-if="doc.outstandingAmount?.float > 0" />
                  <div
                    v-if="doc.outstandingAmount?.float > 0"
                    class="flex justify-between text-red-600 font-semibold text-base"
                  >
                    <div>{{ t`Outstanding Amount` }}</div>
                    <div>{{ formattedValue('outstandingAmount') }}</div>
                  </div>
                </div>
              </div>
            </div>
            <div v-if="doc.items.length" class="mt-5">
                <Button
                class="font-semibold pl-5 pr-3 border border-gray-300 rounded shadow w-full mt-2"
                :icon="true"
                type="secondary"
                @click="clearItems"
                :padding="false"                                   
              >
                <span class="mr-1">{{ t`Clear area` }}</span>                                  
              </Button>        
            </div>
          </template>

          <template #quickedit v-if="quickEditDoc">
            <QuickEditForm
              class="w-quick-edit"
              :name="quickEditDoc.name"
              :show-name="false"
              :show-save="false"
              :source-doc="quickEditDoc"
              :source-fields="quickEditFields"
              :schema-name="quickEditDoc.schemaName"
              :white="true"
              :route-back="false"
              :load-on-close="false"
              @close="toggleQuickEditDoc(null)"
            />
          </template>
      
    </PosContainer>
    
  </template>
  <script>
  import Barcode from 'src/components/Controls/Barcode.vue';
  import { computed } from '@vue/reactivity';
  import { getDocStatus } from 'models/helpers';
  import { ModelNameEnum } from 'models/types';
  import Button from 'src/components/Button.vue';
  import ExchangeRate from 'src/components/Controls/ExchangeRate.vue';
  import FormControl from 'src/components/Controls/FormControl.vue';
  import Table from 'src/components/Controls/Table.vue';
  import DropdownWithActions from 'src/components/DropdownWithActions.vue';
  import PosContainer from 'src/components/PosContainer.vue';
  import StatusBadge from 'src/components/StatusBadge.vue';
  import { fyo } from 'src/initFyo';
  import { docsPathMap } from 'src/utils/misc';
  import {
  docsPath,
  getActionsForDoc,
  routeTo,
  showMessageDialog
  } from 'src/utils/ui';
  import { nextTick } from 'vue';
  import { handleErrorWithDialog } from '../errorHandling';
  import QuickEditForm from './QuickEditForm.vue';
  import Paginator from 'src/components/Paginator.vue';
  
  export default {
    name: 'InvoicePosForm',
    props: { schemaName: String, name: String },
    components: {
      StatusBadge,
      Button,
      FormControl,
      DropdownWithActions,
      Table,
      PosContainer,
      QuickEditForm,
      ExchangeRate,
      Paginator,    
      Barcode
    },
    provide() {
      return {
        schemaName: this.schemaName,
        name: this.name,
        doc: computed(() => this.doc),
      };
    },
    data() {
      return {
        chstatus: false,
        doc: null,
        quickEditDoc: null,
        quickEditFields: [],
        color: null,
        printSettings: null,
        companyName: null,
        items: null,
        searchResults:null,
        pageStart: 0,
        pageEnd: 0,
      };
    },
    updated() {
      this.chstatus = !this.chstatus;
    },
    computed: {
      dataSlice() {
        return this.searchResults? this.searchResults.slice(this.pageStart, this.pageEnd):this.items.slice(this.pageStart, this.pageEnd);
      },
      count() {
        return this.pageEnd - this.pageStart + 1;
      },
      address() {
        return this.printSettings && this.printSettings.getLink('address');
      },
      status() {
        this.chstatus;
        return getDocStatus(this.doc);
      },
      // filtered search results items
      filteredItems(){
        return this.searchResults? this.searchResults: this.items;
      },
      discountNote() {
        const zeroInvoiceDiscount = this.doc?.discountAmount?.isZero();
        const zeroItemDiscount = this.itemDiscountAmount?.isZero();
  
        if (zeroInvoiceDiscount && zeroItemDiscount) {
          return '';
        }
  
        if (!this.doc?.taxes?.length) {
          return '';
        }
  
        let text = this.t`Discount applied before taxation`;
        if (this.doc.discountAfterTax) {
          text = this.t`Discount applied after taxation`;
        }
  
        return text;
      },
      totalDiscount() {
        return this.discountAmount.add(this.itemDiscountAmount);
      },
      discountAmount() {
        return this.doc?.getInvoiceDiscountAmount();
      },
      itemDiscountAmount() {
        return this.doc.getItemDiscountAmount();
      },
      fromCurrency() {
        return this.doc?.currency ?? this.toCurrency;
      },
      toCurrency() {
        return fyo.singles.SystemSettings.currency;
      },
      suggestions() {
      if (!this.searcher) {
        return [];
      }

      const suggestions = this.searcher.search(this.inputValue);
      if (this.limit === -1) {
        return suggestions;
      }

      return suggestions.slice(0, this.limit);
    },
    },

    activated() {
      docsPath.value = docsPathMap[this.schemaName];
    },
    deactivated() {
      docsPath.value = '';
    },
    async mounted() {
      try {
        this.doc = await fyo.doc.getDoc(this.schemaName, this.name);
      } catch (error) {
        if (error instanceof fyo.errors.NotFoundError) {
          routeTo(`/list/${this.schemaName}`);
          return;
        }
        await this.handleError(error);
      }
      this.printSettings = await fyo.doc.getDoc('PrintSettings');
      this.companyName = (await fyo.doc.getDoc('AccountingSettings')).companyName;
  
      let query = this.$route.query;
      if (query.values && query.schemaName === this.schemaName) {
        this.doc.set(this.$router.currentRoute.value.query.values);
      }
  
      if (fyo.store.isDevelopment) {
        window.inv = this;
      }
      this.getItems();
    },
    methods: {
      routeTo,
      toggleInvoiceSettings() {
        if (!this.schemaName) {
          return;
        }
  
        const fields = ['discountAfterTax'].map((fn) =>
          fyo.getField(this.schemaName, fn)
        );
  
        this.toggleQuickEditDoc(this.doc, fields);
      },
      async getItems (){
        this.items = (
          await fyo.db.getAll('Item', {
            fields: ['*']
          })
        );                    
      },
      // setting page indices
      setPageIndices({ start, end }) {
        this.pageStart = start;
        this.pageEnd = end;
      },
      // get item search
      async getSearchInput(){
       if(this.items && this.input!=""){
        let new_items = this.items.filter(item=>{
          return Object.values(item).join(" ").toLowerCase().includes(this.inputValue.toLowerCase());
        });
        this.searchResults = new_items;
       }
      },
      // // clear items from invoice area
      async clearItems(){
        const doc = await fyo.doc.getNewDoc(ModelNameEnum.SalesInvoice, this.filters ?? {});
        const path =  `/pos/SalesInvoice/${doc.name}`;      
        routeTo(path);
        doc.on('afterSync', () => {
          const path =  `/pos/SalesInvoice/${doc.name}`;
          this.$router.replace(path);
        });      
      },
      async addItem(item = null) {    
        let item_doc = await fyo.doc.getDoc("Item", item);      
        this.doc.append("items",{
          item: item_doc.name,
          rate: item_doc.rate,
          tax: item_doc.tax
    });
  
      },
      async toggleQuickEditDoc(doc, fields = []) {
        if (this.quickEditDoc && doc) {
          this.quickEditDoc = null;
          this.quickEditFields = [];
          await nextTick();
        }
  
        this.quickEditDoc = doc;
        this.quickEditFields = fields;
      },
      actions() {
        return getActionsForDoc(this.doc);
      },
      getField(fieldname) {
        return fyo.getField(this.schemaName, fieldname);
      },
      async sync() {
        try {
          await this.doc.sync();
        } catch (err) {
          await this.handleError(err);
        }
      },
      async submit() {
        const message =
          this.schemaName === ModelNameEnum.SalesInvoice
            ? this.t`Submit Sales Invoice?`
            : this.t`Submit Purchase Invoice?`;
        const ref = this;
        await showMessageDialog({
          message,
          buttons: [
            {
              label: this.t`Yes`,
              async action() {
                try {
                  await ref.doc.submit();
                } catch (err) {
                  await ref.handleError(err);
                }
              },
            },
            {
              label: this.t`No`,
              action() {},
            },
          ],
        });
      },
      async handleError(e) {
        await handleErrorWithDialog(e, this.doc);
      },
      formattedValue(fieldname, doc) {
        if (!doc) {
          doc = this.doc;
        }
  
        const df = this.getField(fieldname);
        return fyo.format(doc[fieldname], df, doc);
      },
    },
  };
  </script>
  