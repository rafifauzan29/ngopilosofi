<template>
  <f7-page class="page-bg">
    <f7-navbar title="Pesanan Saya" back-link="Back" class="navbar-custom">
      <f7-nav-right>
        <f7-link icon-only @click="refreshCart" class="refresh-btn">
          <f7-icon ios="f7:arrow_clockwise" aurora="f7:arrow_clockwise" md="material:refresh"></f7-icon>
        </f7-link>
      </f7-nav-right>
    </f7-navbar>

    <f7-block v-if="cartItems.length === 0" class="empty-cart">
      <div class="empty-cart-container">
        <div class="empty-cart-icon">
          <f7-icon ios="f7:cart_fill" aurora="f7:cart_fill" md="material:shopping_cart" size="64px"
            color="#331c2c"></f7-icon>
        </div>
        <div class="empty-title">Keranjang Kosong</div>
        <div class="empty-subtitle">Belum ada item di keranjang Anda</div>
        <f7-button href="/user/menu-list/" class="browse-button">Jelajahi Menu</f7-button>
      </div>
    </f7-block>

    <f7-block v-else class="cart-container">
      <div class="cart-header">
        <f7-row>
          <f7-col>
            <div class="selection-control">
              <f7-checkbox :checked="allSelected" @change="toggleSelectAll" class="custom-checkbox"></f7-checkbox>
              <span class="select-all-text">Pilih Semua</span>
            </div>
          </f7-col>
          <f7-col class="text-align-right">
            <f7-button small @click="removeSelected" class="remove-selected-btn">
              Hapus Terpilih
            </f7-button>
          </f7-col>
        </f7-row>
      </div>

      <div class="cart-items">
        <div v-for="(item, index) in cartItems" :key="index" class="cart-item-card">
          <div class="item-checkbox">
            <f7-checkbox :checked="item.selected" @change="toggleItemSelection(index)"
              class="custom-checkbox"></f7-checkbox>
          </div>
          <div class="item-image" @click="editItem(index)">
            <img :src="item.produk.gambar" />
          </div>
          <div class="item-details" @click="editItem(index)">
            <div class="item-name">{{ item.produk.nama }}</div>
            <div class="item-addons">{{ formatAddons(item.tambahan) }}</div>
            <div class="item-quantity">Jumlah: {{ item.jumlah }}</div>
          </div>
          <div class="item-price-actions">
            <div class="item-price">{{ formatRupiah(item.totalHarga) }}</div>
            <f7-button small @click.stop="removeItem(index)" class="remove-item-btn">
              <f7-icon ios="f7:trash" aurora="f7:trash" md="material:delete"></f7-icon>
            </f7-button>
          </div>
        </div>
      </div>

      <div class="cart-summary">
        <div class="summary-row">
          <span>Total Item Terpilih</span>
          <span class="summary-value">{{ selectedItemsCount }}</span>
        </div>
        <div class="summary-row total-price">
          <span>Total Harga</span>
          <span class="summary-value">{{ formatRupiah(selectedItemsPrice) }}</span>
        </div>
      </div>

      <f7-button large fill round class="checkout-btn" @click="checkout" :disabled="selectedItemsCount === 0">
        <span class="checkout-text">Checkout</span>
        <span class="checkout-price">{{ formatRupiah(selectedItemsPrice) }}</span>
      </f7-button>
    </f7-block>

    <f7-popup v-model:opened="editPopupOpened" class="edit-popup">
      <f7-page class="edit-popup-bg">
        <f7-navbar :title="editingItem ? editingItem.produk.nama : ''" class="popup-navbar">
          <f7-nav-right>
            <f7-link popup-close class="popup-close-btn">Tutup</f7-link>
          </f7-nav-right>
        </f7-navbar>

        <f7-block v-if="editingItem" class="edit-content">
          <div class="edit-image-container">
            <img :src="editingItem.produk.gambar" class="edit-image" />
          </div>

          <div class="edit-section">
            <div class="section-title">Tambahan</div>
            <div class="addons-list">
              <div v-for="(addon, index) in editingItem.produk.tambahan" :key="index" class="addon-item">
                <f7-checkbox :checked="isAddonSelected(addon)" @change="toggleAddon(addon)"
                  class="addon-checkbox"></f7-checkbox>
                <div class="addon-info">
                  <div class="addon-name">{{ addon.nama }}</div>
                  <div class="addon-price">{{ formatRupiah(addon.harga) }}</div>
                </div>
              </div>
            </div>
          </div>

          <div class="edit-section">
            <div class="section-title">Jumlah</div>
            <div class="quantity-control">
              <f7-button small round @click="decreaseEditQuantity" class="quantity-btn minus">
                <f7-icon ios="f7:minus" aurora="f7:minus" md="material:remove"></f7-icon>
              </f7-button>
              <div class="quantity-value">{{ editQuantity }}</div>
              <f7-button small round @click="increaseEditQuantity" class="quantity-btn plus">
                <f7-icon ios="f7:plus" aurora="f7:plus" md="material:add"></f7-icon>
              </f7-button>
            </div>
          </div>

          <f7-button large fill round color="primary" @click="saveEdit" class="save-edit-btn">
            Simpan Perubahan
          </f7-button>
        </f7-block>
      </f7-page>
    </f7-popup>
  </f7-page>
</template>

<script>
import { f7, f7ready } from 'framework7-vue';
import { useCartStore } from '../../js/stores/cart';

export default {
  name: 'OrderPage',
  setup() {
    const cartStore = useCartStore();
    return { cartStore };
  },
  data() {
    return {
      cartItems: [],
      editPopupOpened: false,
      editingItem: null,
      editingIndex: -1,
      editQuantity: 1,
      editSelectedAddons: [],
      isLoading: false,
      isSyncing: false,
      menuItems: []
    };
  },
  computed: {
    totalItems() {
      return this.cartItems.reduce((sum, item) => sum + item.jumlah, 0);
    },
    totalPrice() {
      return this.cartItems.reduce((sum, item) => sum + item.totalHarga, 0);
    },
    selectedItemsCount() {
      return this.cartItems
        .filter(item => item.selected)
        .reduce((sum, item) => sum + item.jumlah, 0);
    },
    selectedItemsPrice() {
      return this.cartItems
        .filter(item => item.selected)
        .reduce((sum, item) => sum + item.totalHarga, 0);
    },
    allSelected() {
      return this.cartItems.length > 0 && this.cartItems.every(item => item.selected);
    }
  },
  methods: {
    formatRupiah(angka) {
      return new Intl.NumberFormat('id-ID', {
        style: 'currency',
        currency: 'IDR',
        minimumFractionDigits: 0
      }).format(angka);
    },
    formatAddons(addons) {
      if (!addons || addons.length === 0) return 'Tanpa tambahan';
      return addons.map(a => a.nama).join(', ');
    },
    async refreshCart() {
      if (this.isSyncing) return;

      this.isSyncing = true;
      try {
        await Promise.all([this.loadMenu(), this.loadCart()]);
        this.showToast('Keranjang diperbarui');
      } catch (error) {
        console.error('Error refreshing cart:', error);
        this.showToast('Gagal memuat keranjang');
      } finally {
        this.isSyncing = false;
      }
    },
    async loadMenu() {
      try {
        const response = await fetch('https://ngopilosofi-production.up.railway.app/api/menu');
        if (!response.ok) throw new Error('Failed to fetch menu');
        this.menuItems = await response.json();
        localStorage.setItem('menuItems', JSON.stringify(this.menuItems));
      } catch (error) {
        console.error('Error loading menu:', error);
        const savedMenu = localStorage.getItem('menuItems');
        this.menuItems = savedMenu ? JSON.parse(savedMenu) : [];
      }
    },
    async loadCart() {
      this.isLoading = true;
      try {
        const token = localStorage.getItem('token');
        if (!token) {
          const savedCart = localStorage.getItem('/user/cart/');
          this.cartItems = savedCart ? JSON.parse(savedCart) : [];
          this.cartStore.count = this.totalItems; 
          return;
        }

        await this.cartStore.fetchCart(); 
        this.cartItems = this.cartStore.items.map(item => {
          const menuItem = this.menuItems.find(m => m._id === item.menuItem._id) || item.menuItem;
          return {
            _id: item._id,
            produk: {
              _id: menuItem._id,
              nama: menuItem.nama,
              harga: menuItem.harga,
              gambar: menuItem.gambar,
              tambahan: menuItem.tambahan || []
            },
            tambahan: item.addons || [],
            jumlah: item.quantity,
            totalHarga: item.totalPrice,
            selected: true
          };
        });

        localStorage.setItem('/user/cart/', JSON.stringify(this.cartItems));

      } catch (error) {
        console.error('Error loading cart:', error);
        const savedCart = localStorage.getItem('/user/cart/');
        this.cartItems = savedCart ? JSON.parse(savedCart) : [];
        this.cartStore.count = this.totalItems; 

        if (navigator.onLine) {
          this.showToast('Gagal memuat keranjang');
        } else {
          this.showToast('Menggunakan data offline');
        }
      } finally {
        this.isLoading = false;
      }
    },
    findCartItemIndex(menuItemId, addons) {
      return this.cartItems.findIndex(item => {
        if (item.produk._id !== menuItemId) return false;
        if (item.tambahan.length !== addons.length) return false;

        const itemAddonIds = item.tambahan.map(a => a._id).sort();
        const newAddonIds = addons.map(a => a._id).sort();

        return JSON.stringify(itemAddonIds) === JSON.stringify(newAddonIds);
      });
    },
    async addToCart(menuItem, quantity = 1, addons = []) {
      try {
        const existingItemIndex = this.findCartItemIndex(menuItem._id, addons);

        if (existingItemIndex !== -1) {
          await this.updateCartItemQuantity(existingItemIndex, this.cartItems[existingItemIndex].jumlah + quantity);
          this.showToast('Jumlah item diperbarui');
          return;
        }

        const token = localStorage.getItem('token');
        const totalAddonPrice = addons.reduce((sum, addon) => sum + addon.harga, 0);
        const totalHarga = (menuItem.harga + totalAddonPrice) * quantity;

        if (!token) {
          const newItem = {
            _id: Date.now().toString(),
            produk: menuItem,
            tambahan: addons,
            jumlah: quantity,
            totalHarga: totalHarga,
            selected: true
          };
          this.cartItems.push(newItem);
          this.saveLocalCart();
          this.showToast('Item ditambahkan ke keranjang (offline)');
          return;
        }

        const response = await fetch('https://ngopilosofi-production.up.railway.app/api/cart', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
          },
          body: JSON.stringify({
            menuItemId: menuItem._id,
            quantity: quantity,
            addons: addons.map(a => a._id)
          })
        });

        if (!response.ok) throw new Error('Failed to add item');

        await this.loadCart();
        this.showToast('Item ditambahkan ke keranjang');

      } catch (error) {
        console.error('Error adding to cart:', error);
        this.showToast('Gagal menambahkan item');
      }
    },
    async updateCartItemQuantity(index, newQuantity) {
      try {
        const item = this.cartItems[index];
        const token = localStorage.getItem('token');

        if (!token) {
          const totalAddonPrice = item.tambahan.reduce((sum, addon) => sum + addon.harga, 0);
          const totalHarga = (item.produk.harga + totalAddonPrice) * newQuantity;

          this.cartItems[index] = {
            ...item,
            jumlah: newQuantity,
            totalHarga: totalHarga
          };
          this.saveLocalCart();
          this.cartStore.count = this.totalItems; 
          return;
        }

        const success = await this.cartStore.updateCartItem(
          item._id,
          newQuantity,
          item.tambahan
        );

        if (!success) throw new Error('Failed to update quantity');

        await this.loadCart();

      } catch (error) {
        console.error('Error updating quantity:', error);
        throw error;
      }
    },
    async removeItem(index) {
      const itemId = this.cartItems[index]._id;

      this.showConfirmDialog(
        'Apakah Anda yakin ingin menghapus item ini?',
        'Hapus Item',
        async () => {
          try {
            const token = localStorage.getItem('token');

            if (!token) {
              this.cartItems.splice(index, 1);
              this.saveLocalCart();
              this.cartStore.count = this.totalItems; 
              this.showToast('Item dihapus dari keranjang');
              return;
            }

            const success = await this.cartStore.removeFromCart(itemId);
            if (!success) throw new Error('Failed to delete item');

            await this.loadCart();
            this.showToast('Item dihapus dari keranjang');

          } catch (error) {
            console.error('Error removing item:', error);
            this.showToast('Gagal menghapus item');
          }
        }
      );
    },
    saveLocalCart() {
      localStorage.setItem('/user/cart/', JSON.stringify(this.cartItems));
      this.$emit('cartUpdated', this.cartItems.length);
    },
    showConfirmDialog(text, title, callback) {
      f7ready(() => {
        f7.dialog.confirm(text, title, callback);
      });
    },
    showToast(message, type = 'normal') {
      f7ready(() => {
        f7.toast.create({
          text: message,
          closeTimeout: 2000,
          destroyOnClose: true,
          cssClass: `custom-toast ${type}`
        }).open();
      });
    },
    editItem(index) {
      this.editingIndex = index;
      this.editingItem = JSON.parse(JSON.stringify(this.cartItems[index]));
      this.editQuantity = this.editingItem.jumlah;
      this.editSelectedAddons = this.editingItem.tambahan.map(addon => {
        const productAddon = this.editingItem.produk.tambahan.find(a => a._id === addon._id);
        return productAddon || addon;
      });

      this.editPopupOpened = true;
    },

    isAddonSelected(addon) {
      return this.editSelectedAddons.some(a => a._id === addon._id);
    },
    isAddonSelected(addon) {
      return this.editSelectedAddons.some(a => a._id === addon._id);
    },
    toggleAddon(addon) {
      const index = this.editSelectedAddons.findIndex(a => a._id === addon._id);
      if (index === -1) {
        this.editSelectedAddons.push(addon);
      } else {
        this.editSelectedAddons.splice(index, 1);
      }
    },
    increaseEditQuantity() {
      this.editQuantity++;
    },
    decreaseEditQuantity() {
      if (this.editQuantity > 1) this.editQuantity--;
    },
    async saveEdit() {
      const totalAddonPrice = this.editSelectedAddons.reduce((sum, addon) => sum + addon.harga, 0);
      const totalHarga = (this.editingItem.produk.harga + totalAddonPrice) * this.editQuantity;

      try {
        const duplicateIndex = this.cartItems.findIndex((item, index) => {
          if (index === this.editingIndex) return false;
          if (item.produk._id !== this.editingItem.produk._id) return false;
          if (item.tambahan.length !== this.editSelectedAddons.length) return false;

          const itemAddonIds = item.tambahan.map(a => a._id).sort();
          const editAddonIds = this.editSelectedAddons.map(a => a._id).sort();

          return JSON.stringify(itemAddonIds) === JSON.stringify(editAddonIds);
        });

        if (duplicateIndex !== -1) {
          const combinedQuantity = this.cartItems[duplicateIndex].jumlah + this.editQuantity;

          if (this.editingIndex > duplicateIndex) {
            await this.removeItem(this.editingIndex);
            await this.updateCartItemQuantity(duplicateIndex, combinedQuantity);
          } else {
            await this.updateCartItemQuantity(duplicateIndex, combinedQuantity);
            await this.removeItem(this.editingIndex);
          }

          this.editPopupOpened = false;
          this.showToast('Item digabungkan dengan yang sudah ada');
          return;
        }

        const token = localStorage.getItem('token');

        if (duplicateIndex !== -1) {
          const duplicateItem = this.cartItems[duplicateIndex];
          const newJumlah = duplicateItem.jumlah + this.editQuantity;

          if (!token) {
            const totalHargaBaru = (duplicateItem.produk.harga +
              this.editSelectedAddons.reduce((sum, a) => sum + a.harga, 0)) * newJumlah;

            this.cartItems[duplicateIndex] = {
              ...duplicateItem,
              jumlah: newJumlah,
              totalHarga: totalHargaBaru
            };
            this.cartItems.splice(this.editingIndex, 1);
            this.saveLocalCart();
            this.showToast('Item digabung (offline)');
            this.editPopupOpened = false;
            return;
          }

          await fetch(`https://ngopilosofi-production.up.railway.app/api/cart/${duplicateItem._id}`, {
            method: 'PUT',
            headers: {
              'Content-Type': 'application/json',
              'Authorization': `Bearer ${token}`
            },
            body: JSON.stringify({
              quantity: newJumlah,
              addons: duplicateItem.tambahan.map(a => a._id)
            })
          });

          await fetch(`https://ngopilosofi-production.up.railway.app/api/cart/${this.editingItem._id}`, {
            method: 'DELETE',
            headers: {
              'Authorization': `Bearer ${token}`
            }
          });

          await this.loadCart();
          this.showToast('Item digabung');
        } else {
          if (!token) {
            const updatedItem = {
              ...this.editingItem,
              jumlah: this.editQuantity,
              tambahan: this.editSelectedAddons,
              totalHarga
            };
            this.cartItems.splice(this.editingIndex, 1, updatedItem);
            this.saveLocalCart();
            this.showToast('Item diperbarui (offline)');
            this.editPopupOpened = false;
            return;
          }

          await fetch(`https://ngopilosofi-production.up.railway.app/api/cart/${this.editingItem._id}`, {
            method: 'PUT',
            headers: {
              'Content-Type': 'application/json',
              'Authorization': `Bearer ${token}`
            },
            body: JSON.stringify({
              quantity: this.editQuantity,
              addons: this.editSelectedAddons.map(a => a._id)
            })
          });

          await this.loadCart();
          this.showToast('Item diperbarui');
        }

        this.editPopupOpened = false;
      } catch (error) {
        console.error('Error saving edit:', error);
        this.showToast('Gagal menyimpan perubahan');
      }
    },
    async checkout() {
      if (!localStorage.getItem('token')) {
        this.showAlert('Silakan login terlebih dahulu', 'Checkout Gagal');
        f7.views.main.router.navigate('/login/');
        return;
      }

      const selectedItems = this.cartItems.filter(item => item.selected);

      if (selectedItems.length === 0) {
        this.showAlert('Silakan pilih minimal 1 item untuk checkout', 'Checkout Gagal');
        return;
      }

      try {
        const token = localStorage.getItem('token');
        const response = await fetch('https://ngopilosofi-production.up.railway.app/api/orders', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
          },
          body: JSON.stringify({
            items: selectedItems.map(item => ({
              menuItemId: item.produk._id,
              quantity: item.jumlah,
              addons: item.tambahan.map(a => a._id),
              price: item.totalHarga
            }))
          })
        });

        if (!response.ok) throw new Error('Checkout failed');

        await fetch('https://ngopilosofi-production.up.railway.app/api/cart', {
          method: 'DELETE',
          headers: {
            'Authorization': `Bearer ${token}`
          }
        });

        const orderData = await response.json();
        localStorage.setItem('/user/checkout/', JSON.stringify({
          orderId: orderData._id,
          items: selectedItems,
          totalItems: this.selectedItemsCount,
          totalPrice: this.selectedItemsPrice,
          timestamp: new Date().toISOString()
        }));

        f7.views.main.router.navigate('/user/order-confirmation/');

      } catch (error) {
        console.error('Checkout error:', error);
        this.showAlert('Gagal melakukan checkout. Silakan coba lagi.', 'Checkout Gagal');
      }
    },
    showAlert(text, title) {
      f7ready(() => {
        f7.dialog.alert(text, title);
      });
    },
    toggleItemSelection(index) {
      this.cartItems[index].selected = !this.cartItems[index].selected;
      this.saveLocalCart();
    },
    toggleSelectAll() {
      const newSelectState = !this.allSelected;
      this.cartItems = this.cartItems.map(item => ({
        ...item,
        selected: newSelectState
      }));
      this.saveLocalCart();
    },
    async removeSelected() {
      if (this.selectedItemsCount === 0) {
        this.showToast('Tidak ada item yang dipilih');
        return;
      }

      this.showConfirmDialog(
        'Apakah Anda yakin ingin menghapus item yang dipilih?',
        'Hapus Item Terpilih',
        async () => {
          try {
            const token = localStorage.getItem('token');

            if (!token) {
              this.cartItems = this.cartItems.filter(item => !item.selected);
              this.saveLocalCart();
              this.cartStore.count = this.totalItems; // Update store
              this.showToast('Item terpilih dihapus (offline)');
              return;
            }

            const selectedIds = this.cartItems
              .filter(item => item.selected)
              .map(item => item._id);

            const success = await this.cartStore.bulkRemoveItems(selectedIds);
            if (!success) throw new Error('Failed to remove items');

            await this.loadCart();
            this.showToast('Item terpilih dihapus');

          } catch (error) {
            console.error('Error removing selected items:', error);
            this.showToast('Gagal menghapus item terpilih');
          }
        }
      );
    },
    setupOnlineHandler() {
      window.addEventListener('online', async () => {
        if (navigator.onLine) {
          try {
            await Promise.all([this.loadMenu(), this.loadCart()]);
            this.showToast('Data disinkronisasi');
          } catch (error) {
            console.error('Error syncing data:', error);
          }
        }
      });
    }
  },
  async mounted() {
    await Promise.all([this.loadMenu(), this.loadCart()]);
    this.setupOnlineHandler();
  }
};
</script>

<style scoped>
.page-bg {
  background-color: #ede0d1;
  font-family: 'Segoe UI', Roboto, 'Helvetica Neue', sans-serif;
}

.navbar-custom {
  background-color: #331c2c;
  color: white;
  box-shadow: 0 2px 10px rgba(108, 92, 231, 0.2);
}

.navbar-custom .left a {
  color: white;
}

.refresh-btn {
  color: white;
}

.empty-cart {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 70vh;
}

.empty-cart-container {
  text-align: center;
  padding: 20px;
}

.empty-cart-icon {
  margin-bottom: 20px;
}

.empty-title {
  font-size: 22px;
  font-weight: 600;
  color: #2d3436;
  margin-bottom: 8px;
}

.empty-subtitle {
  font-size: 16px;
  color: #636e72;
  margin-bottom: 24px;
}

.browse-button {
  --f7-button-bg-color: #331c2c;
  --f7-button-text-color: white;
  --f7-button-font-size: 16px;
  --f7-button-height: 48px;
  border-radius: 12px;
  font-weight: 500;
  box-shadow: 0 4px 12px rgba(108, 92, 231, 0.3);
}

.cart-container {
  padding-bottom: 80px;
}

.cart-header {
  background-color: white;
  padding: 12px 16px;
  margin-bottom: 8px;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.selection-control {
  display: flex;
  align-items: center;
}

.select-all-text {
  font-size: 15px;
  color: #2d3436;
  margin-left: 8px;
}

.remove-selected-btn {
  --f7-button-text-color: #e74c3c;
  --f7-button-border-color: #e74c3c;
  --f7-button-border-width: 1px;
  --f7-button-height: 32px;
  --f7-button-font-size: 13px;
  font-weight: 500;
}

.custom-checkbox {
  --f7-checkbox-active-color: rgb(22, 175, 22);
}

.cart-items {
  margin-bottom: 16px;
}

.cart-item-card {
  display: flex;
  background-color: #ffffff;
  border-radius: 12px;
  padding: 12px;
  margin-bottom: 10px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
  align-items: center;
  transition: transform 0.2s, box-shadow 0.2s;
}

.cart-item-card:active {
  transform: scale(0.98);
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.1);
}

.item-checkbox {
  margin-right: 12px;
}

.item-image {
  width: 70px;
  height: 70px;
  border-radius: 8px;
  overflow: hidden;
  margin-right: 12px;
  flex-shrink: 0;
}

.item-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.item-details {
  flex-grow: 1;
  overflow: hidden;
}

.item-name {
  font-size: 16px;
  font-weight: 600;
  color: #2d3436;
  margin-bottom: 4px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.item-addons {
  font-size: 13px;
  color: #636e72;
  margin-bottom: 4px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.item-quantity {
  font-size: 14px;
  color: #2d3436;
}

.item-price-actions {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  margin-left: 12px;
}

.item-price {
  font-size: 16px;
  font-weight: 600;
  color: #331c2c;
  margin-bottom: 8px;
}

.remove-item-btn {
  --f7-button-text-color: #e74c3c;
  --f7-button-border-color: #e74c3c;
  --f7-button-border-width: 1px;
  --f7-button-height: 28px;
  --f7-button-font-size: 12px;
  min-width: 28px;
  padding: 0 8px;
}

.cart-summary {
  background-color: white;
  border-radius: 12px;
  padding: 16px;
  margin: 20px 0;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.summary-row {
  display: flex;
  justify-content: space-between;
  margin-bottom: 12px;
  font-size: 16px;
  color: #2d3436;
}

.summary-row.total-price {
  font-weight: 600;
  font-size: 18px;
  color: #331c2c;
  margin-bottom: 0;
}

.summary-value {
  font-weight: 500;
}

.checkout-btn {
  --f7-button-bg-color: #331c2c;
  --f7-button-text-color: white;
  --f7-button-height: 40px;
  --f7-button-font-size: 16px;
  font-weight: 600;
  box-shadow: 0 4px 12px rgba(49, 48, 56, 0.3);
  margin-top: 0;
}

.checkout-btn[disabled] {
  --f7-button-bg-color: #b2b2b2;
  box-shadow: none;
}

.checkout-text {
  margin-right: 8px;
}

.edit-popup-bg {
  background-color: #ede0d1;
}

.popup-navbar {
  background-color: #331c2c;
  color: #331c2c;
}

.popup-navbar .right a {
  color: #331c2c;
  font-weight: 500;
}

.popup-close-btn {
  font-size: 16px;
}

.edit-content {
  padding-bottom: 30px;
}

.edit-image-container {
  display: flex;
  justify-content: center;
  margin-bottom: 20px;
}

.edit-image {
  width: 200px;
  height: 200px;
  border-radius: 12px;
  object-fit: cover;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.edit-section {
  background-color: white;
  border-radius: 12px;
  padding: 16px;
  margin-bottom: 16px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.section-title {
  font-size: 16px;
  font-weight: 600;
  color: #2d3436;
  margin-bottom: 12px;
}

.addons-list {
  margin-top: 8px;
}

.addon-item {
  display: flex;
  align-items: center;
  padding: 10px 0;
  border-bottom: 1px solid #f1f2f6;
}

.addon-item:last-child {
  border-bottom: none;
}

.addon-checkbox {
  margin-right: 12px;
}

.addon-info {
  flex-grow: 1;
}

.addon-name {
  font-size: 15px;
  color: #2d3436;
  margin-bottom: 2px;
}

.addon-price {
  font-size: 14px;
  color: #331c2c;
  font-weight: 500;
}

.quantity-control {
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 10px 0;
}

.quantity-btn {
  --f7-button-height: 36px;
  --f7-button-width: 36px;
  --f7-button-text-color: #331c2c;
  --f7-button-border-color: #331c2c;
  --f7-button-border-width: 1px;
}

.quantity-value {
  font-size: 18px;
  font-weight: 600;
  color: #2d3436;
  margin: 0 20px;
  min-width: 24px;
  text-align: center;
}

.save-edit-btn {
  --f7-button-bg-color: #331c2c;
  --f7-button-text-color: white;
  --f7-button-height: 48px;
  --f7-button-font-size: 16px;
  font-weight: 600;
  margin-top: 20px;
  box-shadow: 0 4px 12px rgba(108, 92, 231, 0.3);
}

:root {
  --f7-toast-text-color: white;
  --f7-toast-bg-color: #331c2c;
  --f7-toast-border-radius: 8px;
}
</style>