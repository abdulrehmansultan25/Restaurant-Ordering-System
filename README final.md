
# Project Title

A brief description of what this project does and who it's for

import streamlit as st
import uuid
from datetime import datetime
from abc import ABC, abstractmethod

# ==============================================================================
# 1. CSS FOR SYSTEM-WIDE STYLING (MATCHING THE EMERALD & SLATE DARK THEME)
# ==============================================================================
st.set_page_config(
    page_title="Food Hub - OOP Restaurant Ordering System",
    page_icon="🍔",
    layout="wide",
    initial_sidebar_state="expanded"
)

st.markdown("""
<style>
    /* Global CSS Customizations */
    .stApp {
        background-color: #020617;
        color: #f1f5f9;
    }
    div[data-testid="stSidebar"] {
        background-color: #0f172a;
        border-right: 1px solid #1e293b;
    }
    .main-header {
        font-family: 'Inter', sans-serif;
        font-weight: 800;
        color: #ffffff;
        letter-spacing: -0.05em;
    }
    .badge-oop {
        background-color: #10b981;
        color: #020617;
        padding: 4px 8px;
        border-radius: 8px;
        font-weight: 900;
        font-size: 0.8rem;
    }
    .card-item {
        background-color: #0f172a;
        border: 1px solid #1e293b;
        border-radius: 12px;
        padding: 16px;
        margin-bottom: 12px;
        transition: transform 0.2s ease;
    }
    .card-item:hover {
        border-color: #10b981;
    }
    .log-entry {
        font-family: 'JetBrains Mono', 'Fira Code', monospace;
        font-size: 0.8rem;
        background-color: #020617;
        padding: 8px;
        border-radius: 6px;
        border-left: 3px solid #10b981;
        margin-bottom: 6px;
    }
</style>
""", unsafe_allow_html=True)


# ==============================================================================
# 2. OOP CORE MODEL CLASSES (RESTAURANT DOMAIN)
# ==============================================================================

class OOPLogger:
    """Handles static global system telemetry logs for academic inspection."""
    logs = []

    @classmethod
    def log(cls, title: str, message: str, category: str = "General"):
        timestamp = datetime.now().strftime("%H:%M:%S.%f")[:-3]
        log_entry = {
            "timestamp": timestamp,
            "title": title,
            "message": message,
            "category": category
        }
        cls.logs.insert(0, log_entry)  # Insert at top for live feed representation


class MenuItem:
    """Represents an encapsulated Menu item within the system."""
    def __init__(self, item_id: str, name: str, description: str, price: float, category: str, image_url: str, is_available: bool = True, spicy_level: str = "Mild"):
        self.id = item_id
        self.name = name
        self.description = description
        self.price = price
        self.category = category
        self.image_url = image_url
        self.is_available = is_available
        self.spicy_level = spicy_level


class CartItem:
    """Represents an item instantiation within the user's active Cart."""
    def __init__(self, menu_item: MenuItem, quantity: int, customizations: dict = None):
        self.menu_item = menu_item
        self.quantity = quantity
        self.customizations = customizations if customizations else {"extra_cheese": False, "instructions": ""}
        self.unit_price = menu_item.price
        self.recalculate_totals()

    def recalculate_totals(self):
        base = self.unit_price
        if self.customizations.get("extra_cheese"):
            base += 40.0  # Extra cheese cost
        self.total_price = base * self.quantity


class Cart:
    """Represents a State Container managing active selections."""
    def __init__(self):
        self.items = []

    def add_menu_item(self, menu_item: MenuItem, quantity: int, customizations: dict):
        # Look for existing identical configuration in cart
        for item in self.items:
            if item.menu_item.id == menu_item.id and item.customizations == customizations:
                item.quantity += quantity
                item.recalculate_totals()
                OOPLogger.log("Cart Action", f"Incremented quantity for '{menu_item.name}' (Qty: {item.quantity})", "Cart")
                return
        
        # Or instantiate a new CartItem
        new_item = CartItem(menu_item, quantity, customizations)
        self.items.append(new_item)
        OOPLogger.log("Cart Action", f"Added item '{menu_item.name}' to basket", "Cart")

    def remove_item(self, index: int):
        if 0 <= index < len(self.items):
            removed = self.items.pop(index)
            OOPLogger.log("Cart Action", f"Removed '{removed.menu_item.name}' from basket", "Cart")

    def update_quantity(self, index: int, quantity: int):
        if 0 <= index < len(self.items):
            if quantity <= 0:
                self.remove_item(index)
            else:
                self.items[index].quantity = quantity
                self.items[index].recalculate_totals()
                OOPLogger.log("Cart Action", f"Updated quantity for '{self.items[index].menu_item.name}' to {quantity}", "Cart")

    def calculate_subtotal(self) -> float:
        return sum(item.total_price for item in self.items)

    def calculate_final_total(self) -> float:
        subtotal = self.calculate_subtotal()
        # Delivery Fee is Rs. 99, Tax is 5%
        if subtotal == 0:
            return 0.0
        return subtotal + 99.0 + (subtotal * 0.05)

    def clear(self):
        self.items = []
        OOPLogger.log("Cart Action", "Cart cleared successfully", "Cart")


# ==============================================================================
# 3. POLYMORPHIC STRATEGY: PAYMENT PROCESSORS
# ==============================================================================

class PaymentProcessor(ABC):
    """Abstract Base Class defining the polymorphic interface for Payments."""
    @abstractmethod
    def process_payment(self, amount: float) -> dict:
        pass


class CashPaymentProcessor(PaymentProcessor):
    def process_payment(self, amount: float) -> dict:
        OOPLogger.log("Payment Success", f"Polymorphic Execution: Cash On Delivery Selected. Authorized Rs. {amount}", "Payment")
        return {"success": True, "transaction_id": "CASH-" + str(uuid.uuid4())[:8].upper(), "gateway": "CashGateway"}


class CardPaymentProcessor(PaymentProcessor):
    def __init__(self, card_number: str, expiry: str):
        self.card_number = card_number
        self.expiry = expiry

    def process_payment(self, amount: float) -> dict:
        masked_card = f"****-****-****-{self.card_number[-4:] if len(self.card_number) >= 4 else '1111'}"
        OOPLogger.log("Payment Success", f"Polymorphic Execution: Processed Card {masked_card}. Transferred Rs. {amount}", "Payment")
        return {"success": True, "transaction_id": "CARD-" + str(uuid.uuid4())[:8].upper(), "gateway": "BankAlfalah_v3"}


class EasyPaisaPaymentProcessor(PaymentProcessor):
    def __init__(self, wallet_num: str):
        self.wallet_num = wallet_num

    def process_payment(self, amount: float) -> dict:
        OOPLogger.log("Payment Success", f"Polymorphic Execution: EasyPaisa Wallet {self.wallet_num} Charged Rs. {amount}", "Payment")
        return {"success": True, "transaction_id": "EP-" + str(uuid.uuid4())[:8].upper(), "gateway": "EasyPaisa_API_v2"}


# ==============================================================================
# 4. ORDER DISPATCH & INHERITANCE
# ==============================================================================

class Order:
    """Represents a sealed, finalized transaction Order instance."""
    def __init__(self, order_id: str, cart_items: list, subtotal: float, final_total: float, customer_info: dict, payment_info: dict):
        self.order_id = order_id
        self.items = cart_items
        self.subtotal = subtotal
        self.tax = subtotal * 0.05
        self.delivery_fee = 99.0
        self.final_total = final_total
        self.customer_name = customer_info.get("name")
        self.customer_phone = customer_info.get("phone")
        self.delivery_address = customer_info.get("address")
        self.payment_method = payment_info.get("method")
        self.transaction_id = payment_info.get("transaction_id")
        self.payment_status = "Paid" if self.payment_method != "Cash on Delivery" else "Pending (Cash)"
        self.order_status = "Pending"  # "Pending" -> "Preparing" -> "Out for Delivery" -> "Delivered"
        self.timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        self.logs = [f"[{self.timestamp}] Order created and queued for processing."]

    def advance_status(self):
        status_flow = ["Pending", "Preparing", "Out for Delivery", "Delivered"]
        try:
            curr_idx = status_flow.index(self.order_status)
            if curr_idx < len(status_flow) - 1:
                self.order_status = status_flow[curr_idx + 1]
                time_str = datetime.now().strftime("%H:%M:%S")
                log_msg = f"[{time_str}] Order transitioned to: {self.order_status}"
                self.logs.append(log_msg)
                OOPLogger.log("Order State Shift", f"Order {self.order_id} advanced to '{self.order_status}'", "Dispatch")
        except ValueError:
            pass


# ==============================================================================
# 5. SINGLETON DESIGN PATTERN: RESTAURANT MANAGER
# ==============================================================================

class RestaurantManager:
    """Singleton Controller wrapping database and transaction pipelines."""
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super(RestaurantManager, cls).__new__(cls)
            cls._instance.is_initialized = False
        return cls._instance

    def initialize(self):
        if self.is_initialized:
            return
        
        # Seed Menu Data (Exactly matching application dataset)
        self.menu_catalog = [
            MenuItem("F1", "Fettuccine Alfredo", "Rich parmesan cream sauce, grilled chicken medallions, and flat noodles.", 590, "mains", "https://images.unsplash.com/photo-1645112411341-6c4fd023714a?auto=format&fit=crop&q=80&w=400"),
            MenuItem("F2", "Classic Beef Burger", "Premium smashed beef patty, caramelized onions, cheddar slice, and artisan brioche bun.", 450, "mains", "https://images.unsplash.com/photo-1568901346375-23c9450c58cd?auto=format&fit=crop&q=80&w=400"),
            MenuItem("F3", "Cheesy Garlic Breadsticks", "Baked golden crust brushed with garlic butter and loaded with gooey strings of premium Mozzarella.", 250, "starters", "https://images.unsplash.com/photo-1573145959986-a111a69027ef?auto=format&fit=crop&q=80&w=400"),
            MenuItem("F4", "Choco Lava Cake", "Hot chocolate cake with a molten chocolate reservoir served with premium vanilla scoop.", 320, "desserts", "https://images.unsplash.com/photo-1606313564200-e75d5e30476c?auto=format&fit=crop&q=80&w=400"),
            MenuItem("F5", "Mint Margarita", "Refreshing blend of fresh mint, lime juice, brown sugar, and sparkling club soda.", 180, "beverages", "https://images.unsplash.com/photo-1513558161293-cdaf765ed2fd?auto=format&fit=crop&q=80&w=400")
        ]
        
        self.current_cart = Cart()
        self.orders_history = {}
        
        OOPLogger.log("System Initialization", "Restaurant Manager Singleton instantiated successfully.", "General")
        self.is_initialized = True


# Instantiate the singleton in session state so it persists across stream renders
if "manager" not in st.session_state:
    manager_inst = RestaurantManager()
    manager_inst.initialize()
    st.session_state["manager"] = manager_inst
else:
    manager_inst = st.session_state["manager"]


# ==============================================================================
# 6. STREAMLIT FRONT-END INTERFACE & PORTALS
# ==============================================================================

# Custom Sidebar
with st.sidebar:
    st.markdown(f"""
    <div style="display: flex; align-items: center; gap: 10px; margin-bottom: 20px;">
        <div class="badge-oop">FH</div>
        <div>
            <h2 style="margin: 0; font-size: 1.4rem;" class="main-header">Food Hub</h2>
            <span style="font-size: 0.65rem; color: #94a3b8; font-family: monospace;">v1.2-OOP-Python</span>
        </div>
    </div>
    """, unsafe_allow_html=True)
    
    st.markdown("""---""")
    
    portal_selector = st.radio(
        "Select Portal View:",
        ["🛒 Browse & Order", "🎓 OOP Assessment Portal"],
        index=0
    )
    
    st.markdown("""---""")
    
    st.markdown("""
    **Academic QuickBadge**
    <div style="background-color: #020617; border: 1px solid #1e293b; padding: 12px; border-radius: 8px;">
        <span style="font-size: 0.75rem; color: #10b981; font-family: monospace; font-weight: bold;">ABDUL REHMAN SULTAN</span><br/>
        <span style="font-size: 0.65rem; color: #64748b; font-family: monospace;">BS BME (Riphah)</span>
    </div>
    """, unsafe_allow_html=True)


# PORTAL A: RESTAURANT ORDERING SYSTEM
if portal_selector == "🛒 Browse & Order":
    st.markdown("<h1 class='main-header'>Browse Menu Catalog & Order</h1>", unsafe_allow_html=True)
    st.write("Browse premium culinary crafts, customize options, and secure polymorphic checkouts.")

    order_tab1, order_tab2, order_tab3 = st.tabs(["🍽️ Catalog & Basket", "💳 Secure Checkout", "🚚 Order Tracking Status"])

    # TAB 1: MENU CATALOG AND ACTIVE BASKET
    with order_tab1:
        col1, col2 = st.columns([2, 1])

        with col1:
            st.markdown("### Menu Catalog")
            category_filter = st.selectbox("Filter Category:", ["All", "Starters", "Mains", "Desserts", "Beverages"])
            
            for item in manager_inst.menu_catalog:
                # Apply filter
                if category_filter != "All" and item.category != category_filter.lower():
                    continue

                with st.container():
                    st.markdown(f"""
                    <div class="card-item">
                        <div style="display: flex; justify-content: space-between;">
                            <h4>{item.name}</h4>
                            <span style="color: #10b981; font-family: monospace; font-weight: bold;">Rs. {item.price}</span>
                        </div>
                        <p style="font-size: 0.85rem; color: #94a3b8;">{item.description}</p>
                        <span style="font-size: 0.7rem; color: #475569; font-family: monospace;">Spiciness: {item.spicy_level} | Category: {item.category.capitalize()}</span>
                    </div>
                    """, unsafe_allow_html=True)
                    
                    # Customizations Controls
                    cust_col1, cust_col2, cust_col3 = st.columns([1, 1, 1])
                    with cust_col1:
                        extra_cheese = st.checkbox("Extra Cheese (+Rs. 40)", key=f"cheese_{item.id}")
                    with cust_col2:
                        qty = st.number_input("Quantity:", min_value=1, max_value=10, value=1, key=f"qty_{item.id}")
                    with cust_col3:
                        if st.button("Add to Basket", key=f"add_{item.id}", use_container_width=True):
                            customizations = {"extra_cheese": extra_cheese, "instructions": "Standard preparation"}
                            manager_inst.current_cart.add_menu_item(item, qty, customizations)
                            st.toast(f"Added {item.name} x {qty} to basket!", icon="🛒")
                            st.rerun()

        with col2:
            st.markdown("### Active Basket")
            cart = manager_inst.current_cart
            if len(cart.items) == 0:
                st.info("Your shopping basket is currently empty. Choose delicacies from the catalog!")
            else:
                for idx, cart_item in enumerate(cart.items):
                    with st.container():
                        st.markdown(f"""
                        <div style="background-color: #020617; border: 1px solid #334155; padding: 10px; border-radius: 8px; margin-bottom: 8px;">
                            <div style="display: flex; justify-content: space-between;">
                                <strong style="font-size: 0.85rem;">{cart_item.menu_item.name}</strong>
                                <span style="font-size: 0.85rem; color: #10b981; font-family: monospace;">Rs. {cart_item.total_price}</span>
                            </div>
                            <span style="font-size: 0.7rem; color: #64748b; font-family: monospace;">
                                Qty: {cart_item.quantity} x Rs. {cart_item.unit_price} 
                                {'(+ Extra Cheese)' if cart_item.customizations['extra_cheese'] else ''}
                            </span>
                        </div>
                        """, unsafe_allow_html=True)
                        
                        del_col, space_col = st.columns([1, 3])
                        with del_col:
                            if st.button("Remove", key=f"del_{idx}"):
                                cart.remove_item(idx)
                                st.rerun()
                
                # Receipt Calculations
                st.markdown("---")
                subtotal = cart.calculate_subtotal()
                tax = subtotal * 0.05
                delivery = 99.0
                total = cart.calculate_final_total()

                st.markdown(f"""
                <div style="font-family: monospace; font-size: 0.85rem; background-color: #0f172a; padding: 12px; border-radius: 8px; border: 1px dashed #475569;">
                    <div style="display: flex; justify-content: space-between;"><span>Subtotal:</span><span>Rs. {subtotal}</span></div>
                    <div style="display: flex; justify-content: space-between;"><span>Tax (5%):</span><span>Rs. {tax:.1f}</span></div>
                    <div style="display: flex; justify-content: space-between;"><span>Delivery:</span><span>Rs. {delivery}</span></div>
                    <hr style="border-top: 1px dashed #475569; margin: 8px 0;"/>
                    <div style="display: flex; justify-content: space-between; font-weight: bold; color: #10b981; font-size: 1rem;">
                        <span>FINAL TOTAL:</span><span>Rs. {total:.1f}</span>
                    </div>
                </div>
                """, unsafe_allow_html=True)

    # TAB 2: SECURE CHECKOUT (POLYMORPHIC STRATEGY SELECTION)
    with order_tab2:
        st.markdown("### Secure Checkout Portal")
        cart = manager_inst.current_cart
        
        if len(cart.items) == 0:
            st.warning("Please add items to your basket before initiating checkout!")
        else:
            checkout_form = st.form(key="payment_form")
            with checkout_form:
                st.markdown("##### 1. Delivery Coordinates")
                cust_name = st.text_input("Full Name:", value="Abdul Rehman Sultan")
                cust_phone = st.text_input("Mobile Number:", value="0300-1234567")
                cust_address = st.text_area("Delivery Address:", value="Riphah International University, Islamabad")

                st.markdown("##### 2. Polymorphic Payment Strategy Selection")
                pay_method = st.selectbox(
                    "Choose Payment Method:",
                    ["Cash on Delivery", "Credit/Debit Card", "EasyPaisa Mobile Wallet"]
                )

                card_num = ""
                card_exp = ""
                ep_num = ""

                if pay_method == "Credit/Debit Card":
                    col_card1, col_card2 = st.columns(2)
                    with col_card1:
                        card_num = st.text_input("Card Number:", value="1234-5678-9876-5432")
                    with col_card2:
                        card_exp = st.text_input("Expiration Date:", value="12/29")
                elif pay_method == "EasyPaisa Mobile Wallet":
                    ep_num = st.text_input("EasyPaisa Account Account Number:", value="0312-3456789")

                submit_order = st.form_submit_button("Seal Order & Pay Rs. " + str(cart.calculate_final_total()))

                if submit_order:
                    # Resolve Polymorphic Payment Strategy
                    processor = None
                    if pay_method == "Cash on Delivery":
                        processor = CashPaymentProcessor()
                    elif pay_method == "Credit/Debit Card":
                        processor = CardPaymentProcessor(card_num, card_exp)
                    elif pay_method == "EasyPaisa Mobile Wallet":
                        processor = EasyPaisaPaymentProcessor(ep_num)

                    # Process Payment Polymorphically
                    final_amount = cart.calculate_final_total()
                    payment_receipt = processor.process_payment(final_amount)

                    # Instantiate Order object
                    order_id = "FH-" + str(uuid.uuid4())[:6].upper()
                    new_order = Order(
                        order_id=order_id,
                        cart_items=list(cart.items),
                        subtotal=cart.calculate_subtotal(),
                        final_total=final_amount,
                        customer_info={"name": cust_name, "phone": cust_phone, "address": cust_address},
                        payment_info={"method": pay_method, "transaction_id": payment_receipt["transaction_id"]}
                    )

                    # Save Order object into historical records
                    manager_inst.orders_history[order_id] = new_order
                    OOPLogger.log("Order Processed", f"Successfully recorded Order {order_id}", "Database")
                    
                    # Clear Cart State
                    cart.clear()
                    st.success(f"Receipt Generated! Order ID: **{order_id}** processed via polymorphic context.")
                    st.toast("Redirecting to tracker...", icon="📦")
                    st.balloons()

    # TAB 3: ORDER DISPATCH TRACKER
    with order_tab3:
        st.markdown("### Live Order Dispatch Status")
        if len(manager_inst.orders_history) == 0:
            st.info("No orders placed during this session. Finish checkout to trigger live tracking.")
        else:
            selected_order_id = st.selectbox("Select Order to Track:", list(manager_inst.orders_history.keys()))
            order = manager_inst.orders_history[selected_order_id]

            col_track1, col_track2 = st.columns([2, 1])

            with col_track1:
                st.markdown(f"#### Active Tracked Order ID: `{order.order_id}`")
                
                # Graphical representation of State transition
                statuses = ["Pending", "Preparing", "Out for Delivery", "Delivered"]
                curr_status = order.order_status
                
                status_cols = st.columns(4)
                for i, status in enumerate(statuses):
                    with status_cols[i]:
                        is_done = statuses.index(curr_status) >= i
                        color = "🟢" if is_done else "⚪"
                        st.markdown(f"**{color} {status}**")
                
                # Manual state driver for simulating lifecycle actions
                st.markdown("---")
                if curr_status != "Delivered":
                    if st.button("Simulate Next Transit Stage"):
                        order.advance_status()
                        st.rerun()
                else:
                    st.success("🎉 This order has been successfully dispatched and delivered!")

            with col_track2:
                st.markdown("#### Transaction Receipt")
                st.markdown(f"""
                <div style="font-family: monospace; font-size: 0.8rem; background-color: #0f172a; padding: 15px; border-radius: 8px; border: 1px solid #1e293b;">
                    <strong>Customer:</strong> {order.customer_name}<br/>
                    <strong>Phone:</strong> {order.customer_phone}<br/>
                    <strong>Address:</strong> {order.delivery_address}<br/>
                    <hr style="border-top: 1px dashed #334155; margin: 8px 0;"/>
                    <strong>Payment Strategy:</strong> {order.payment_method}<br/>
                    <strong>Transaction Hash:</strong> {order.transaction_id}<br/>
                    <strong>Payment State:</strong> {order.payment_status}<br/>
                    <hr style="border-top: 1px dashed #334155; margin: 8px 0;"/>
                    <strong>Receipt Breakdown:</strong><br/>
                    Mains & Starters: Rs. {order.subtotal:.2f}<br/>
                    GST Tax Amount: Rs. {order.tax:.2f}<br/>
                    Courier Delivery: Rs. {order.delivery_fee:.2f}<br/>
                    <strong>Grand Total: Rs. {order.final_total:.2f}</strong>
                </div>
                """, unsafe_allow_html=True)


# PORTAL B: OOP ACADEMIC DIAGNOSTICS DASHBOARD
elif portal_selector == "🎓 OOP Assessment Portal":
    st.markdown("<h1 class='main-header'>OOP Assessment Portal</h1>", unsafe_allow_html=True)
    st.write("Live structural telemetry, architecture diagnostics, and diagnostic log analysis for Abdul Rehman Sultan.")

    col_diag1, col_diag2 = st.columns([1, 1])

    with col_diag1:
        st.markdown("### System Log Stream")
        for entry in OOPLogger.logs:
            st.markdown(f"""
            <div class="log-entry">
                <span style="color: #64748b;">[{entry['timestamp']}]</span>
                <span style="color: #10b981; font-weight: bold;">[{entry['category']}]</span>
                <strong>{entry['title']}</strong>: {entry['message']}
            </div>
            """, unsafe_allow_html=True)

    with col_diag2:
        st.markdown("### Interactive OOP State Inspector")
        st.markdown("Active Heap allocation and Singleton states:")
        
        # Singleton State Info
        st.info(f"RestaurantManager Singleton Memory Reference: `{id(manager_inst)}`")
        
        st.markdown("##### Current System Instances Matrix")
        metrics_col1, metrics_col2, metrics_col3 = st.columns(3)
        with metrics_col1:
            st.metric("Catalog Size", f"{len(manager_inst.menu_catalog)} Items")
        with metrics_col2:
            st.metric("Cart Size", f"{len(manager_inst.current_cart.items)} Items")
        with metrics_col3:
            st.metric("Orders Dispatched", f"{len(manager_inst.orders_history)} Sealed")

        st.markdown("---")
        st.markdown("### UML Structural Architecture Map")
        st.markdown("""
        ```text
        +-----------------------------------------------------------+
        |                 RestaurantManager (Singleton)             |
        | - _instance: RestaurantManager                            |
        | + menu_catalog: List[MenuItem]                            |
        | + current_cart: Cart                                      |
        | + orders_history: Dict[str, Order]                        |
        +-----------------------------------------------------------+
                                     |
                                     v
        +----------------------------+------------------------------+
        |                            |                              |
        v                            v                              v
  +------------+               +------------+                +-------------+
  |  MenuItem  |               |    Cart    |                |    Order    |
  | - id, name |               | - items    |                | - id, status|
  | - price    |               +------------+                +-------------+
  +------------+
        ```
        """)

        st.markdown("""
        **Core Design Patterns Implemented:**
        1. **Singleton Pattern**: Built in the `RestaurantManager` class via customized pythonic instance allocation.
        2. **Strategy Pattern**: Implemented polymorphically inside `PaymentProcessor` abstractions (Cash, Card, EasyPaisa models).
        3. **Encapsulation**: Structured data properties and custom methods in `Cart` and `CartItem` ensuring pure OOP compliance.
        """)