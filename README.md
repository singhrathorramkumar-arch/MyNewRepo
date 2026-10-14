import React, { useEffect, useMemo, useState } from 'react';
import { ActivityIndicator, Alert, Image, SafeAreaView, ScrollView, StyleSheet, Text, TextInput, TouchableOpacity, View } from 'react-native';
import { StatusBar } from 'expo-status-bar';
import { Ionicons } from '@expo/vector-icons';
import paymentQr from './assets/payment-qr.png';
import { backendConfigured, supabase } from './lib/supabase';

const NAVY = '#082B52';
const GOLD = '#D8A12B';
const LIGHT = '#F5F7FA';
const initialMaterials = [
  { id: 'sand', name: 'Sand', unit: 'Ton', unit_price: 5500, icon: 'layers-outline' },
  { id: 'concrete', name: 'Concrete', unit: 'm³', unit_price: 2200, icon: 'cube-outline' },
  { id: 'crusher', name: 'Stone Crusher', unit: 'Ton', unit_price: 2000, icon: 'grid-outline' },
  { id: 'gitti', name: 'Gitti / Aggregate', unit: 'Ton', unit_price: 1800, icon: 'ellipse-outline' },
  { id: 'bricks', name: 'Bricks', unit: 'Piece', unit_price: 12, icon: 'apps-outline' },
];
const initialBusiness = {
  id: 1, business_name: 'Meera Ventures', phone: '+917987254906',
  email: 'meera.ventures83@gmail.com', delivery_fee: 5000, advance_percent: 30,
};
const money = value => `₹${Number(value || 0).toLocaleString('en-IN')}`;
const notify = (title, message) => Alert.alert(title, message);

function Field({ label, value, onChangeText, placeholder, secureTextEntry, keyboardType, multiline }) {
  return <View style={styles.fieldWrap}><Text style={styles.label}>{label}</Text><TextInput
    style={[styles.input, multiline && styles.multiline]} value={value} onChangeText={onChangeText}
    placeholder={placeholder || label} secureTextEntry={secureTextEntry} keyboardType={keyboardType || 'default'}
    autoCapitalize={secureTextEntry || keyboardType === 'email-address' ? 'none' : 'sentences'} multiline={multiline}
  /></View>;
}
function Button({ title, onPress, secondary, disabled }) {
  return <TouchableOpacity disabled={disabled} onPress={onPress} style={[styles.button, secondary && styles.secondaryButton, disabled && styles.disabledButton]}>
    <Text style={[styles.buttonText, secondary && styles.secondaryButtonText]}>{title}</Text>
  </TouchableOpacity>;
}
function Header({ title, onBack }) {
  return <View style={styles.header}>{onBack ? <TouchableOpacity onPress={onBack} style={styles.back}><Ionicons name="arrow-back" size={22} color="#fff" /></TouchableOpacity> : null}<Text style={styles.headerTitle}>{title}</Text></View>;
}

export default function App() {
  const [screen, setScreen] = useState('home');
  const [session, setSession] = useState(null);
  const [mode, setMode] = useState('customer');
  const [authKind, setAuthKind] = useState('customer');
  const [authAction, setAuthAction] = useState('signin');
  const [returnScreen, setReturnScreen] = useState('home');
  const [busy, setBusy] = useState(true);
  const [working, setWorking] = useState(false);
  const [fullName, setFullName] = useState('');
  const [phone, setPhone] = useState('');
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [materials, setMaterials] = useState(initialMaterials);
  const [business, setBusiness] = useState(initialBusiness);
  const [cart, setCart] = useState([]);
  const [orders, setOrders] = useState([]);
  const [selected, setSelected] = useState(initialMaterials[2]);
  const [quantity, setQuantity] = useState(1);
  const [deliveryAddress, setDeliveryAddress] = useState('');
  const [deliveryDate, setDeliveryDate] = useState('');
  const [paymentReference, setPaymentReference] = useState('');
  const [firmOrders, setFirmOrders] = useState([]);
  const [businessDraft, setBusinessDraft] = useState(initialBusiness);
  const [profileDraft, setProfileDraft] = useState({ full_name: '', phone: '' });

  useEffect(() => {
    if (!supabase) { setBusy(false); return undefined; }
    let active = true;
    supabase.auth.getSession().then(({ data, error }) => {
      if (!active) return;
      if (error) notify('Sign-in error', error.message);
      setSession(data?.session || null);
      setBusy(false);
    });
    const { data: { subscription } } = supabase.auth.onAuthStateChange((_event, nextSession) => {
      setSession(nextSession);
      setBusy(false);
    });
    return () => { active = false; subscription.unsubscribe(); };
  }, []);

  useEffect(() => {
    let active = true;
    const load = async () => {
      if (!supabase) return;
      const [{ data: productRows, error: productError }, { data: settingsRow, error: settingsError }] = await Promise.all([
        supabase.from('materials').select('id,name,unit,unit_price,icon,active').order('sort_order'),
        supabase.from('business_settings').select('*').eq('id', 1).maybeSingle(),
      ]);
      if (!active) return;
      if (productError) notify('Could not load materials', productError.message);
      if (settingsError) notify('Could not load business details', settingsError.message);
      if (productRows?.length) setMaterials(productRows.filter(item => item.active));
      if (settingsRow) { setBusiness(settingsRow); setBusinessDraft(settingsRow); }
      if (!session?.user?.id) { setCart([]); setOrders([]); setFirmOrders([]); return; }
      const { data: profile } = await supabase.from('profiles').select('full_name,phone').eq('user_id', session.user.id).maybeSingle();
      if (active && profile) setProfileDraft({ full_name: profile.full_name || '', phone: profile.phone || '' });
      if (mode === 'firm') {
        const { data } = await supabase.from('orders').select('*').order('created_at', { ascending: false });
        if (active) setFirmOrders(data || []);
      } else {
        const [{ data: cartRows }, { data: orderRows }] = await Promise.all([
          supabase.from('cart_items').select('id,user_id,material_id,quantity,material:materials(id,name,unit,unit_price,icon)').order('updated_at', { ascending: false }),
          supabase.from('orders').select('*').order('created_at', { ascending: false }),
        ]);
        if (active) { setCart(cartRows || []); setOrders(orderRows || []); }
      }
    };
    load();
    return () => { active = false; };
  }, [session?.user?.id, mode, screen]);

  const subtotal = useMemo(() => cart.reduce((sum, row) => sum + Number(row.material?.unit_price || 0) * row.quantity, 0), [cart]);
  const total = subtotal + Number(business.delivery_fee || 0);
  const advance = Math.round(total * Number(business.advance_percent || 0) / 100);
  const user = session?.user;

  const go = next => {
    if (['cart', 'orders'].includes(next) && !user) {
      setReturnScreen(next); setAuthKind('customer'); setAuthAction('signin'); setScreen('auth'); return;
    }
    setScreen(next);
  };
  const loadCart = async () => {
    if (!supabase || !user) return;
    const { data, error } = await supabase.from('cart_items').select('id,user_id,material_id,quantity,material:materials(id,name,unit,unit_price,icon)').order('updated_at', { ascending: false });
    if (error) notify('Cart error', error.message); else setCart(data || []);
  };
  const addToCart = async () => {
    if (!user) { setReturnScreen('cart'); setAuthKind('customer'); setAuthAction('signin'); setScreen('auth'); return; }
    setWorking(true);
    const { error } = await supabase.from('cart_items').upsert({ user_id: user.id, material_id: selected.id, quantity }, { onConflict: 'user_id,material_id' });
    setWorking(false);
    if (error) return notify('Could not add item', error.message);
    await loadCart(); setScreen('cart');
  };
  const setCartQuantity = async (row, nextQuantity) => {
    if (!supabase) return;
    const result = nextQuantity <= 0
      ? await supabase.from('cart_items').delete().eq('id', row.id)
      : await supabase.from('cart_items').update({ quantity: nextQuantity, updated_at: new Date().toISOString() }).eq('id', row.id);
    if (result.error) notify('Cart error', result.error.message); else await loadCart();
  };

  const authenticate = async () => {
    if (!backendConfigured) return notify('Backend setup needed', 'Configure Supabase using the setup files before creating accounts or signing in.');
    if (!email.trim() || !password) return notify('Missing details', 'Enter your email and password.');
    setWorking(true);
    try {
      if (authAction === 'signup') {
        if (!fullName.trim()) throw new Error('Enter your name.');
        const { data, error } = await supabase.auth.signUp({
          email: email.trim(), password,
          options: { data: { full_name: fullName.trim(), phone: phone.trim() } },
        });
        if (error) throw error;
        if (!data.session) {
          setAuthAction('signin');
          notify('Check your email', 'Confirm your email address, then sign in. Customer accounts cannot create firm access.');
        } else { setMode('customer'); setScreen(returnScreen); }
      } else {
        const { error } = await supabase.auth.signInWithPassword({ email: email.trim(), password });
        if (error) throw error;
        if (authKind === 'firm') {
          const { data: allowed, error: roleError } = await supabase.rpc('is_firm_staff');
          if (roleError || !allowed) {
            await supabase.auth.signOut();
            throw new Error('This account is not authorized for firm access. Ask the firm administrator to grant staff access.');
          }
          setMode('firm'); setScreen('firm');
        } else { setMode('customer'); setScreen(returnScreen); }
      }
    } catch (error) { notify('Could not continue', error.message || 'Please try again.'); }
    finally { setWorking(false); }
  };
  const signOut = async () => {
    if (supabase) await supabase.auth.signOut();
    setSession(null); setMode('customer'); setScreen('home');
  };
  const beginFirmLogin = async () => {
    if (user) await signOut();
    setAuthKind('firm'); setAuthAction('signin'); setScreen('auth');
  };
  const createOrder = async () => {
    if (!deliveryAddress.trim()) return notify('Delivery address needed', 'Enter the delivery address for this order.');
    setWorking(true);
    const { data, error } = await supabase.rpc('create_order_from_cart', {
      p_delivery_address: deliveryAddress.trim(),
      p_delivery_date: deliveryDate.trim() || null,
      p_payment_reference: paymentReference.trim(),
    });
    setWorking(false);
    if (error) return notify('Could not submit order', error.message);
    const record = Array.isArray(data) ? data[0] : data;
    setScreen('orders');
    setDeliveryAddress(''); setDeliveryDate(''); setPaymentReference('');
    notify('Order submitted', `Order ${record?.order_number || ''} is awaiting payment verification. The QR payment is not verified automatically; Meera Ventures must confirm receipt.`);
  };
  const updateOrder = async (order, patch) => {
    const { error } = await supabase.from('orders').update(patch).eq('id', order.id);
    if (error) return notify('Could not update order', error.message);
    const { data } = await supabase.from('orders').select('*').order('created_at', { ascending: false });
    setFirmOrders(data || []);
  };
  const saveBusiness = async () => {
    const { error } = await supabase.from('business_settings').update({
      business_name: businessDraft.business_name.trim(), phone: businessDraft.phone.trim(), email: businessDraft.email.trim(),
      delivery_fee: Number(businessDraft.delivery_fee), advance_percent: Number(businessDraft.advance_percent), updated_at: new Date().toISOString(),
    }).eq('id', 1);
    if (error) return notify('Could not save business details', error.message);
    setBusiness(businessDraft); notify('Saved', 'Business details and checkout defaults were updated.');
  };
  const saveProfile = async () => {
    if (!user || !supabase) return;
    if (!profileDraft.full_name.trim()) return notify('Name required', 'Enter your name before saving.');
    setWorking(true);
    const { error } = await supabase.from('profiles').upsert({
      user_id: user.id, full_name: profileDraft.full_name.trim(), phone: profileDraft.phone.trim(),
    }, { onConflict: 'user_id' });
    setWorking(false);
    if (error) return notify('Could not save profile', error.message);
    notify('Profile saved', 'Your account details have been updated.');
  };
  const setBusinessField = (key, value) => setBusinessDraft(previous => ({ ...previous, [key]: value }));

  const nav = () => mode === 'firm' ? null : <View style={styles.nav}>
    {[['home', 'Home', 'home-outline'], ['cart', `Cart (${cart.length})`, 'cart-outline'], ['orders', 'Orders', 'receipt-outline'], ['account', user ? 'Account' : 'Sign in', 'person-outline']].map(([id, label, icon]) =>
      <TouchableOpacity key={id} style={styles.navItem} onPress={() => id === 'account' && !user ? (setAuthKind('customer'), setAuthAction('signin'), setScreen('auth')) : go(id)}>
        <Ionicons name={icon} size={20} color={screen === id ? GOLD : '#6D7A8A'} /><Text style={[styles.navText, screen === id && styles.activeNav]}>{label}</Text>
      </TouchableOpacity>)}
  </View>;
  const orderCard = (order, firm = false) => <View key={order.id} style={styles.card}>
    <View style={styles.orderHead}><Text style={styles.rowTitle}>{order.order_number}</Text><Text style={styles.badge}>{order.status}</Text></View>
    {firm ? <><Text style={styles.bodyText}>{order.customer_name} · {order.customer_email}</Text><Text style={styles.bodyText}>{order.customer_phone}</Text><Text style={styles.bodyText}>Deliver to: {order.delivery_address}</Text></> : null}
    {(order.items || []).map((item, index) => <Text key={`${order.id}-${index}`} style={styles.bodyText}>{item.name} · {item.quantity} {item.unit} · {money(item.line_total)}</Text>)}
    <Text style={styles.bodyText}>Total {money(order.total)} · Advance {money(order.advance_amount)}</Text>
    <Text style={styles.muted}>Payment: {order.payment_status}{order.payment_reference ? ` · Ref ${order.payment_reference}` : ''}</Text>
    {firm ? <View style={styles.actionRow}>
      {order.payment_status !== 'confirmed' && <Button title="Confirm payment" onPress={() => updateOrder(order, { payment_status: 'confirmed', status: 'Confirmed' })} />}
      {order.payment_status === 'submitted' && <Button title="Reject payment" secondary onPress={() => updateOrder(order, { payment_status: 'rejected' })} />}
      {['Confirmed', 'Preparing', 'Out for delivery'].includes(order.status) && <Button title={order.status === 'Confirmed' ? 'Mark preparing' : order.status === 'Preparing' ? 'Out for delivery' : 'Mark delivered'} secondary onPress={() => updateOrder(order, { status: ({ Confirmed: 'Preparing', Preparing: 'Out for delivery', 'Out for delivery': 'Delivered' })[order.status] })} />}
    </View> : null}
  </View>;

  if (busy) return <SafeAreaView style={styles.center}><ActivityIndicator size="large" color={NAVY} /><Text style={styles.muted}>Loading account…</Text></SafeAreaView>;
  if (screen === 'auth') return <SafeAreaView style={styles.safe}><StatusBar style="dark" /><Header title={authKind === 'firm' ? 'Firm staff sign in' : authAction === 'signup' ? 'Create customer account' : 'Customer sign in'} onBack={() => setScreen('account')} />
    <ScrollView contentContainerStyle={styles.page}>
      {!backendConfigured ? <View style={styles.setupNotice}><Text style={styles.noticeTitle}>Connect the account backend first</Text><Text style={styles.bodyText}>Set up the Supabase project using supabase/schema.sql, then add the project URL and publishable key to a local .env file.</Text></View> : null}
      {authKind === 'firm' ? <Text style={styles.muted}>Firm access is granted by the business administrator; customer registration cannot create staff access.</Text> : authAction === 'signup' ? <><Field label="Full name" value={fullName} onChangeText={setFullName} /><Field label="Phone" value={phone} onChangeText={setPhone} keyboardType="phone-pad" /></> : null}
      <Field label="Email" value={email} onChangeText={setEmail} keyboardType="email-address" />
      <Field label="Password" value={password} onChangeText={setPassword} secureTextEntry />
      <Button title={working ? 'Please wait…' : authAction === 'signup' ? 'Create customer account' : 'Sign in'} onPress={authenticate} disabled={working || !backendConfigured} />
      {authKind === 'customer' ? <Button secondary title={authAction === 'signup' ? 'Already have an account? Sign in' : 'New customer? Create account'} onPress={() => setAuthAction(authAction === 'signup' ? 'signin' : 'signup')} /> : null}
      {authKind === 'customer' ? <Button secondary title="Firm staff sign in" onPress={() => { setAuthKind('firm'); setAuthAction('signin'); }} /> : <Button secondary title="Customer sign in" onPress={() => { setAuthKind('customer'); setAuthAction('signin'); }} />}
    </ScrollView>{nav()}</SafeAreaView>;

  if (screen === 'home') return <SafeAreaView style={styles.safe}><StatusBar style="light" backgroundColor={NAVY} /><ScrollView>
    <View style={styles.hero}><Text style={styles.logo}>MV</Text><Text style={styles.brand}>MEERA VENTURES</Text><Text style={styles.tag}>Building Materials, Delivered to Your Site.</Text></View>
    <Text style={styles.section}>What do you need?</Text><View style={styles.grid}>{materials.map(item => <TouchableOpacity key={item.id} style={styles.tile} onPress={() => { setSelected(item); setQuantity(1); setScreen('product'); }}><Ionicons name={item.icon || 'cube-outline'} size={30} color={NAVY} /><Text style={styles.tileText}>{item.name}</Text><Text style={styles.price}>{money(item.unit_price)} / {item.unit}</Text></TouchableOpacity>)}</View>
    <Button title="View materials" onPress={() => setScreen('materials')} />
    <View style={styles.info}><Text style={styles.infoTitle}>Your account</Text><Text style={styles.infoText}>{user ? `Signed in as ${user.email}` : 'Create an account to save your cart and view order history.'}</Text><Button secondary title={user ? 'Open account' : 'Sign in or create account'} onPress={() => user ? setScreen('account') : (setAuthKind('customer'), setAuthAction('signin'), setScreen('auth'))} /></View>
  </ScrollView>{nav()}</SafeAreaView>;

  if (screen === 'materials') return <SafeAreaView style={styles.safe}><StatusBar style="dark" /><Header title="Construction Materials" onBack={() => setScreen('home')} /><ScrollView contentContainerStyle={styles.page}>{materials.map(item => <TouchableOpacity key={item.id} style={styles.row} onPress={() => { setSelected(item); setQuantity(1); setScreen('product'); }}><Ionicons name={item.icon || 'cube-outline'} size={28} color={NAVY} /><View style={styles.flex}><Text style={styles.rowTitle}>{item.name}</Text><Text style={styles.muted}>{money(item.unit_price)} / {item.unit}</Text></View><Ionicons name="chevron-forward" size={20} color="#8993A0" /></TouchableOpacity>)}</ScrollView>{nav()}</SafeAreaView>;

  if (screen === 'product') return <SafeAreaView style={styles.safe}><StatusBar style="dark" /><Header title="Product details" onBack={() => setScreen('materials')} /><ScrollView contentContainerStyle={styles.page}><View style={styles.productImg}><Ionicons name={selected.icon || 'cube-outline'} size={70} color={NAVY} /></View><Text style={styles.bigTitle}>{selected.name}</Text><Text style={styles.muted}>Price: {money(selected.unit_price)} / {selected.unit}</Text>
    <Text style={styles.label}>Quantity</Text><View style={styles.stepper}><TouchableOpacity onPress={() => setQuantity(Math.max(1, quantity - 1))}><Text style={styles.step}>−</Text></TouchableOpacity><Text style={styles.qty}>{quantity} {selected.unit}</Text><TouchableOpacity onPress={() => setQuantity(quantity + 1)}><Text style={styles.step}>+</Text></TouchableOpacity></View>
    <Text style={styles.bodyText}>Item total: {money(Number(selected.unit_price) * quantity)}</Text><Button title={working ? 'Adding…' : 'Add to saved cart'} onPress={addToCart} disabled={working} />
  </ScrollView>{nav()}</SafeAreaView>;

  if (screen === 'cart') return <SafeAreaView style={styles.safe}><StatusBar style="dark" /><Header title="My cart" onBack={() => setScreen('home')} /><ScrollView contentContainerStyle={styles.page}>
    {!user ? <><Text style={styles.bodyText}>Sign in to keep your cart saved to your account.</Text><Button title="Sign in" onPress={() => { setAuthKind('customer'); setAuthAction('signin'); setReturnScreen('cart'); setScreen('auth'); }} /></> : null}
    {cart.map(row => <View key={row.id} style={styles.cartRow}><View style={styles.flex}><Text style={styles.rowTitle}>{row.material?.name}</Text><Text style={styles.muted}>{money(row.material?.unit_price)} / {row.material?.unit}</Text><Text style={styles.bodyText}>{money(Number(row.material?.unit_price) * row.quantity)}</Text></View><View style={styles.qtyControls}><TouchableOpacity onPress={() => setCartQuantity(row, row.quantity - 1)}><Text style={styles.step}>−</Text></TouchableOpacity><Text style={styles.qty}>{row.quantity}</Text><TouchableOpacity onPress={() => setCartQuantity(row, row.quantity + 1)}><Text style={styles.step}>+</Text></TouchableOpacity></View></View>)}
    {user && cart.length === 0 ? <Text style={styles.bodyText}>Your saved cart is empty. Add materials to get started.</Text> : null}
    {cart.length > 0 ? <View style={styles.summary}><Text>Items: {money(subtotal)}</Text><Text>Delivery: {money(business.delivery_fee)}</Text><Text style={styles.totalLabel}>Total: {money(total)}</Text><Button title="Continue to checkout" onPress={() => setScreen('checkout')} /></View> : null}
  </ScrollView>{nav()}</SafeAreaView>;

  if (screen === 'checkout') return <SafeAreaView style={styles.safe}><StatusBar style="dark" /><Header title="Checkout & payment" onBack={() => setScreen('cart')} /><ScrollView contentContainerStyle={styles.page}>
    {cart.map(row => <View key={row.id} style={styles.orderLine}><Text style={styles.bodyText}>{row.material?.name} · {row.quantity} {row.material?.unit}</Text><Text style={styles.bodyText}>{money(Number(row.material?.unit_price) * row.quantity)}</Text></View>)}
    <View style={styles.advance}><Text>Order total: {money(total)}</Text><Text style={styles.advanceAmt}>Advance ({business.advance_percent}%): {money(advance)}</Text></View>
    <Field label="Delivery address" value={deliveryAddress} onChangeText={setDeliveryAddress} multiline />
    <Field label="Preferred delivery date (optional)" value={deliveryDate} onChangeText={setDeliveryDate} placeholder="YYYY-MM-DD" />
    <Field label="Payment reference / UTR (optional)" value={paymentReference} onChangeText={setPaymentReference} placeholder="Enter after paying" />
    <View style={styles.qrCard}><Text style={styles.rowTitle}>{business.business_name}</Text><Text style={styles.muted}>TID: 36945926</Text><Image source={paymentQr} style={styles.qrImage} resizeMode="contain" /><Text style={styles.paymentHint}>Scan the QR and pay the advance amount above. The app cannot verify transfers automatically; the firm will confirm payment after checking its account.</Text></View>
    <Button title={working ? 'Submitting…' : 'Submit order for firm confirmation'} onPress={createOrder} disabled={working || !backendConfigured} />
  </ScrollView>{nav()}</SafeAreaView>;

  if (screen === 'orders') return <SafeAreaView style={styles.safe}><StatusBar style="dark" /><Header title="My order history" onBack={() => setScreen('home')} /><ScrollView contentContainerStyle={styles.page}>{orders.length ? orders.map(order => orderCard(order)) : <Text style={styles.bodyText}>No orders yet. Your orders will appear here after checkout.</Text>}</ScrollView>{nav()}</SafeAreaView>;

  if (screen === 'account') return <SafeAreaView style={styles.safe}><StatusBar style="dark" /><Header title="Customer account" onBack={() => setScreen('home')} /><ScrollView contentContainerStyle={styles.page}>
    {user ? <><Text style={styles.bigTitle}>My account</Text><Text style={styles.bodyText}>{user.email}</Text><Field label="Full name" value={profileDraft.full_name} onChangeText={value => setProfileDraft(previous => ({ ...previous, full_name: value }))} /><Field label="Phone" value={profileDraft.phone} onChangeText={value => setProfileDraft(previous => ({ ...previous, phone: value }))} keyboardType="phone-pad" /><Button title={working ? 'Saving…' : 'Save account details'} onPress={saveProfile} disabled={working} /><Text style={styles.muted}>Your cart and orders are saved to this account.</Text><Button title="View saved cart" onPress={() => setScreen('cart')} /><Button secondary title="View order history" onPress={() => setScreen('orders')} /><Button secondary title="Sign out" onPress={signOut} /><Button secondary title="Firm staff sign in" onPress={beginFirmLogin} /></> : <><Text style={styles.bodyText}>Sign in or create a customer account to save your cart and order history.</Text><Button title="Sign in" onPress={() => { setAuthKind('customer'); setAuthAction('signin'); setScreen('auth'); }} /><Button secondary title="Create customer account" onPress={() => { setAuthKind('customer'); setAuthAction('signup'); setScreen('auth'); }} /><Button secondary title="Firm staff sign in" onPress={beginFirmLogin} /></>}
  </ScrollView>{nav()}</SafeAreaView>;

  if (screen === 'firm') return <SafeAreaView style={styles.safe}><StatusBar style="dark" /><Header title="Firm dashboard" /><ScrollView contentContainerStyle={styles.page}>
    <Text style={styles.bigTitle}>Order summary</Text><View style={styles.summaryTiles}><View style={styles.summaryTile}><Text style={styles.tileText}>{firmOrders.length}</Text><Text style={styles.muted}>Total orders</Text></View><View style={styles.summaryTile}><Text style={styles.tileText}>{firmOrders.filter(order => order.payment_status === 'submitted').length}</Text><Text style={styles.muted}>Payments to review</Text></View><View style={styles.summaryTile}><Text style={styles.tileText}>{firmOrders.filter(order => order.status !== 'Delivered').length}</Text><Text style={styles.muted}>Open orders</Text></View></View>
    <Text style={styles.section}>Orders</Text>{firmOrders.length ? firmOrders.map(order => orderCard(order, true)) : <Text style={styles.bodyText}>No customer orders yet.</Text>}
    <Text style={styles.section}>Business details</Text><View style={styles.card}><Field label="Business name" value={businessDraft.business_name} onChangeText={value => setBusinessField('business_name', value)} /><Field label="Business phone" value={businessDraft.phone} onChangeText={value => setBusinessField('phone', value)} keyboardType="phone-pad" /><Field label="Business email" value={businessDraft.email} onChangeText={value => setBusinessField('email', value)} keyboardType="email-address" /><Field label="Delivery fee (₹)" value={String(businessDraft.delivery_fee)} onChangeText={value => setBusinessField('delivery_fee', value)} keyboardType="numeric" /><Field label="Advance percent" value={String(businessDraft.advance_percent)} onChangeText={value => setBusinessField('advance_percent', value)} keyboardType="numeric" /><Button title="Save business details" onPress={saveBusiness} /></View>
    <Button secondary title="Sign out" onPress={signOut} />
  </ScrollView></SafeAreaView>;

  return <SafeAreaView style={styles.safe}><Header title="Page not found" onBack={() => setScreen('home')} /></SafeAreaView>;
}

const styles = StyleSheet.create({
  safe: { flex: 1, backgroundColor: LIGHT }, center: { flex: 1, alignItems: 'center', justifyContent: 'center', backgroundColor: LIGHT, gap: 12 },
  page: { padding: 16, paddingBottom: 100 }, header: { minHeight: 58, backgroundColor: NAVY, flexDirection: 'row', alignItems: 'center', paddingHorizontal: 16, gap: 12 }, back: { padding: 5 }, headerTitle: { color: '#fff', fontSize: 18, fontWeight: '800', flex: 1 },
  hero: { backgroundColor: NAVY, padding: 34, alignItems: 'center', borderBottomLeftRadius: 26, borderBottomRightRadius: 26 }, logo: { borderWidth: 2, borderColor: GOLD, color: GOLD, fontWeight: '900', fontSize: 28, padding: 8 }, brand: { color: '#fff', fontSize: 26, fontWeight: '900', letterSpacing: 1.2, marginTop: 10 }, tag: { color: '#DDE7F2', marginTop: 7, textAlign: 'center' }, section: { fontSize: 20, fontWeight: '800', color: NAVY, paddingVertical: 16 },
  grid: { flexDirection: 'row', flexWrap: 'wrap', paddingHorizontal: 8 }, tile: { backgroundColor: '#fff', width: '46%', margin: '2%', padding: 16, borderRadius: 14, alignItems: 'center', elevation: 2 }, tileText: { fontWeight: '800', color: NAVY, marginTop: 8, textAlign: 'center' }, price: { fontSize: 12, color: '#6E7782', marginTop: 4 },
  button: { backgroundColor: NAVY, marginVertical: 7, padding: 14, borderRadius: 10, alignItems: 'center', justifyContent: 'center' }, buttonText: { color: '#fff', fontWeight: '800', fontSize: 15, textAlign: 'center' }, secondaryButton: { backgroundColor: '#fff', borderWidth: 1, borderColor: NAVY }, secondaryButtonText: { color: NAVY }, disabledButton: { opacity: 0.5 },
  info: { margin: 18, backgroundColor: '#FFF8E7', padding: 18, borderRadius: 12 }, infoTitle: { fontWeight: '800', color: NAVY }, infoText: { color: '#5E6570', marginTop: 5, lineHeight: 20 }, nav: { minHeight: 66, backgroundColor: '#fff', borderTopWidth: 1, borderTopColor: '#E3E7EC', flexDirection: 'row', justifyContent: 'space-around', alignItems: 'center' }, navItem: { alignItems: 'center', paddingHorizontal: 4 }, navText: { fontSize: 10, color: '#6D7A8A', marginTop: 3 }, activeNav: { color: NAVY, fontWeight: '800' },
  row: { backgroundColor: '#fff', padding: 16, borderRadius: 12, marginBottom: 10, flexDirection: 'row', alignItems: 'center', gap: 13, elevation: 1 }, flex: { flex: 1 }, rowTitle: { fontSize: 16, fontWeight: '800', color: NAVY }, muted: { color: '#75808D', marginTop: 4 }, bodyText: { color: '#293542', marginTop: 7, lineHeight: 21 }, productImg: { height: 170, backgroundColor: '#E8EDF3', borderRadius: 15, alignItems: 'center', justifyContent: 'center', marginBottom: 18 }, bigTitle: { fontSize: 22, fontWeight: '900', color: NAVY, marginTop: 7 }, label: { fontWeight: '800', color: NAVY, marginTop: 14, marginBottom: 7 }, fieldWrap: { marginBottom: 8 }, input: { backgroundColor: '#fff', borderRadius: 10, padding: 13, borderWidth: 1, borderColor: '#E0E5EA', color: '#17212B' }, multiline: { minHeight: 78, textAlignVertical: 'top' }, stepper: { backgroundColor: '#fff', borderRadius: 12, padding: 10, flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between', marginBottom: 12 }, step: { fontSize: 28, color: NAVY, paddingHorizontal: 18 }, qty: { fontWeight: '800' },
  cartRow: { backgroundColor: '#fff', padding: 14, borderRadius: 12, marginBottom: 10, flexDirection: 'row', alignItems: 'center' }, qtyControls: { flexDirection: 'row', alignItems: 'center' }, summary: { backgroundColor: '#fff', padding: 16, borderRadius: 12, marginTop: 18, gap: 8 }, totalLabel: { fontWeight: '900', fontSize: 16, color: NAVY, marginTop: 5 }, orderLine: { flexDirection: 'row', justifyContent: 'space-between', gap: 12, paddingVertical: 5 }, advance: { backgroundColor: '#FFF3D5', padding: 16, borderRadius: 12, marginTop: 12, gap: 6 }, advanceAmt: { fontSize: 18, fontWeight: '900', color: '#A66E00' }, qrCard: { backgroundColor: '#fff', padding: 16, borderRadius: 12, marginTop: 12, alignItems: 'center', gap: 6 }, qrImage: { width: 300, height: 300, alignSelf: 'center', marginTop: 8 }, paymentHint: { color: '#5E6570', margin: 8, lineHeight: 20, textAlign: 'center' }, setupNotice: { backgroundColor: '#FFF3D5', padding: 16, borderRadius: 12, marginBottom: 12 }, noticeTitle: { fontWeight: '900', color: NAVY, marginBottom: 4 },
  summaryTiles: { flexDirection: 'row', flexWrap: 'wrap', gap: 8 }, summaryTile: { flexGrow: 1, backgroundColor: '#fff', padding: 14, borderRadius: 12, minWidth: '30%', alignItems: 'center' }, card: { backgroundColor: '#fff', padding: 16, borderRadius: 12, marginBottom: 12, gap: 6 }, orderHead: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', gap: 8 }, badge: { backgroundColor: '#DDF4E5', color: '#19713B', paddingHorizontal: 8, paddingVertical: 4, borderRadius: 8, fontWeight: '800', fontSize: 11 }, actionRow: { gap: 4 },
});
