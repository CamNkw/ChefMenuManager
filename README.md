# ChefMenuManager


import React, { useEffect, useMemo, useState } from 'react';
import {
  Alert,
  KeyboardAvoidingView,
  Modal,
  Platform,
  Pressable,
  ScrollView,
  StatusBar,
  StyleSheet,
  Text,
  TextInput,
  View,
} from 'react-native';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { SafeAreaView } from 'react-native-safe-area-context';

// --------------------------------------------------
// 1. TYPES AND CONSTANTS
// --------------------------------------------------

type Course = 'Starter' | 'Main Course' | 'Dessert';

interface MenuItem {
  id: string;
  name: string;
  description: string;
  category: Course;
  price: number;
}

const STORAGE_KEY = '@chefs_menu_manager_items';

const CATEGORIES: Course[] = [
  'Starter',
  'Main Course',
  'Dessert',
];

const INITIAL_ITEMS: MenuItem[] = [
  {
    id: '1',
    name: 'Roasted Tomato Soup',
    description: 'Fresh tomatoes with herbs and cream.',
    category: 'Starter',
    price: 65,
  },
  {
    id: '2',
    name: 'Grilled Chicken',
    description: 'Grilled chicken served with vegetables.',
    category: 'Main Course',
    price: 145,
  },
  {
    id: '3',
    name: 'Chocolate Lava Cake',
    description: 'Warm chocolate cake with a soft centre.',
    category: 'Dessert',
    price: 75,
  },
];

// --------------------------------------------------
// 2. MAIN APPLICATION
// --------------------------------------------------

export default function HomeScreen() {
  const [menuItems, setMenuItems] = useState<MenuItem[]>([]);
  const [isLoaded, setIsLoaded] = useState(false);

  const [searchText, setSearchText] = useState('');
  const [selectedCategory, setSelectedCategory] =
    useState<Course | 'All'>('All');

  const [modalVisible, setModalVisible] = useState(false);
  const [editingId, setEditingId] = useState<string | null>(null);

  const [dishName, setDishName] = useState('');
  const [dishDescription, setDishDescription] = useState('');
  const [dishPrice, setDishPrice] = useState('');
  const [dishCategory, setDishCategory] =
    useState<Course>('Starter');

  // ------------------------------------------------
  // 3. LOAD MENU ITEMS FROM ASYNC STORAGE
  // ------------------------------------------------

  useEffect(() => {
    async function loadMenu() {
      try {
        const savedMenu = await AsyncStorage.getItem(STORAGE_KEY);

        if (savedMenu !== null) {
          const parsedMenu: unknown = JSON.parse(savedMenu);

          if (Array.isArray(parsedMenu)) {
            setMenuItems(parsedMenu as MenuItem[]);
          } else {
            setMenuItems(INITIAL_ITEMS);
          }
        } else {
          setMenuItems(INITIAL_ITEMS);
        }
      } catch (error) {
        console.error('Error loading menu:', error);
        setMenuItems(INITIAL_ITEMS);
        Alert.alert(
          'Storage error',
          'The saved menu could not be loaded. Sample dishes have been displayed.'
        );
      } finally {
        setIsLoaded(true);
      }
    }

    loadMenu();
  }, []);

  // ------------------------------------------------
  // 4. SAVE MENU ITEMS AUTOMATICALLY
  // ------------------------------------------------

  useEffect(() => {
    if (!isLoaded) return;

    async function saveMenu() {
      try {
        await AsyncStorage.setItem(
          STORAGE_KEY,
          JSON.stringify(menuItems)
        );
      } catch (error) {
        console.error('Error saving menu:', error);
        Alert.alert(
          'Save error',
          'Your menu could not be saved. Please try again.'
        );
      }
    }

    saveMenu();
  }, [menuItems, isLoaded]);

  // ------------------------------------------------
  // 5. AUTOMATIC MENU STATISTICS
  // ------------------------------------------------

  const statistics = useMemo(() => {
    const totalItems = menuItems.length;

    const starters = menuItems.filter(
      item => item.category === 'Starter'
    ).length;

    const mainCourses = menuItems.filter(
      item => item.category === 'Main Course'
    ).length;

    const desserts = menuItems.filter(
      item => item.category === 'Dessert'
    ).length;

    const totalPrice = menuItems.reduce(
      (total, item) => total + item.price,
      0
    );

    const averagePrice =
      totalItems > 0 ? totalPrice / totalItems : 0;

    return {
      totalItems,
      starters,
      mainCourses,
      desserts,
      averagePrice,
    };
  }, [menuItems]);

  // ------------------------------------------------
  // 6. SEARCH AND CATEGORY FILTERING
  // ------------------------------------------------

  const filteredItems = useMemo(() => {
    return menuItems.filter(item => {
      const matchesSearch =
        item.name.toLowerCase().includes(searchText.trim().toLowerCase());

      const matchesCategory =
        selectedCategory === 'All' ||
        item.category === selectedCategory;

      return matchesSearch && matchesCategory;
    });
  }, [menuItems, searchText, selectedCategory]);

  // ------------------------------------------------
  // 7. RESET FORM
  // ------------------------------------------------

  function resetForm() {
    setDishName('');
    setDishDescription('');
    setDishPrice('');
    setDishCategory('Starter');
    setEditingId(null);
  }

  // ------------------------------------------------
  // 8. OPEN ADD MENU ITEM FORM
  // ------------------------------------------------

  function openAddForm() {
    resetForm();
    setModalVisible(true);
  }

  // ------------------------------------------------
  // 9. OPEN EDIT MENU ITEM FORM
  // ------------------------------------------------

  function openEditForm(item: MenuItem) {
    setEditingId(item.id);
    setDishName(item.name);
    setDishDescription(item.description);
    setDishPrice(item.price.toString());
    setDishCategory(item.category);
    setModalVisible(true);
  }

  // ------------------------------------------------
  // 10. SAVE NEW OR UPDATED MENU ITEM
  // ------------------------------------------------

  function saveMenuItem() {
    const cleanName = dishName.trim();
    const cleanDescription = dishDescription.trim();
    const numericPrice = Number(dishPrice);

    if (!cleanName) {
      Alert.alert('Missing information', 'Please enter a dish name.');
      return;
    }

    if (!cleanDescription) {
      Alert.alert(
        'Missing information',
        'Please enter a dish description.'
      );
      return;
    }

    if (
      dishPrice.trim() === '' ||
      !Number.isFinite(numericPrice) ||
      numericPrice <= 0
    ) {
      Alert.alert(
        'Invalid price',
        'Please enter a valid price greater than zero.'
      );
      return;
    }

    if (editingId) {
      setMenuItems(previousItems =>
        previousItems.map(item =>
          item.id === editingId
            ? {
                ...item,
                name: cleanName,
                description: cleanDescription,
                category: dishCategory,
                price: numericPrice,
              }
            : item
        )
      );

      Alert.alert('Success', 'Menu item updated successfully.');
    } else {
      const newItem: MenuItem = {
        id: `${Date.now()}-${Math.random()}`,
        name: cleanName,
        description: cleanDescription,
        category: dishCategory,
        price: numericPrice,
      };

      setMenuItems(previousItems => [...previousItems, newItem]);

      Alert.alert('Success', 'New menu item added successfully.');
    }

    setModalVisible(false);
    resetForm();
  }

  // ------------------------------------------------
  // 11. DELETE MENU ITEM
  // ------------------------------------------------

  function deleteMenuItem(id: string) {
    Alert.alert(
      'Delete menu item',
      'Are you sure you want to delete this dish?',
      [
        {
          text: 'Cancel',
          style: 'cancel',
        },
        {
          text: 'Delete',
          style: 'destructive',
          onPress: () => {
            setMenuItems(previousItems =>
              previousItems.filter(item => item.id !== id)
            );
          },
        },
      ]
    );
  }

  // ------------------------------------------------
  // 12. CLEAR SEARCH AND FILTERS
  // ------------------------------------------------

  function clearFilters() {
    setSearchText('');
    setSelectedCategory('All');
  }

  // ------------------------------------------------
  // 13. DASHBOARD USER INTERFACE
  // ------------------------------------------------

  return (
    <SafeAreaView style={styles.container}>
      <StatusBar barStyle="dark-content" />

      <ScrollView
        showsVerticalScrollIndicator={false}
        contentContainerStyle={styles.scrollContent}
      >
        {/* HEADER */}

        <View style={styles.header}>
          <View>
            <Text style={styles.greeting}>WELCOME BACK</Text>
            <Text style={styles.title}>Chef's Menu</Text>
            <Text style={styles.subtitle}>Menu Manager</Text>
          </View>

          <View style={styles.profileIcon}>
            <Text style={styles.profileText}>👨‍🍳</Text>
          </View>
        </View>

        {/* WELCOME BANNER */}

        <View style={styles.banner}>
          <Text style={styles.bannerLabel}>
            YOUR KITCHEN AT A GLANCE
          </Text>

          <Text style={styles.bannerTitle}>
            Create something delicious!
          </Text>

          <Text style={styles.bannerDescription}>
            Manage your menu, organise your dishes and keep track of prices.
          </Text>

          <Text style={styles.bannerEmoji}>🍴</Text>
        </View>

        {/* STATISTICS */}

        <View style={styles.sectionHeader}>
          <Text style={styles.sectionTitle}>Menu Overview</Text>
          <Text style={styles.sectionSubtitle}>
            Automatically updated statistics
          </Text>
        </View>

        <View style={styles.statsGrid}>
          <StatCard
            emoji="🍽️"
            title="Total Items"
            value={statistics.totalItems.toString()}
          />

          <StatCard
            emoji="🥗"
            title="Starters"
            value={statistics.starters.toString()}
          />

          <StatCard
            emoji="🍲"
            title="Main Courses"
            value={statistics.mainCourses.toString()}
          />

          <StatCard
            emoji="🍰"
            title="Desserts"
            value={statistics.desserts.toString()}
          />

          <StatCard
            emoji="💰"
            title="Average Price"
            value={`R${statistics.averagePrice.toFixed(2)}`}
            fullWidth
          />
        </View>

        {/* MENU MANAGEMENT */}

        <View style={styles.menuHeader}>
          <View>
            <Text style={styles.sectionTitle}>Manage Your Menu</Text>
            <Text style={styles.sectionSubtitle}>
              Add, edit and organise your dishes
            </Text>
          </View>
        </View>

        <Pressable
          style={styles.addButton}
          onPress={openAddForm}
        >
          <Text style={styles.addButtonText}>
            + Add New Menu Item
          </Text>
        </Pressable>

        {/* SEARCH */}

        <Text style={styles.fieldLabel}>Search menu items</Text>

        <TextInput
          style={styles.searchInput}
          placeholder="Search by dish name..."
          placeholderTextColor="#999999"
          value={searchText}
          onChangeText={setSearchText}
        />

        {/* FILTERS */}

        <Text style={styles.fieldLabel}>Filter by course</Text>

        <View style={styles.filterContainer}>
          {(['All', ...CATEGORIES] as const).map(category => (
            <Pressable
              key={category}
              onPress={() => setSelectedCategory(category)}
              style={[
                styles.filterButton,
                selectedCategory === category &&
                  styles.activeFilterButton,
              ]}
            >
              <Text
                style={[
                  styles.filterText,
                  selectedCategory === category &&
                    styles.activeFilterText,
                ]}
              >
                {category}
              </Text>
            </Pressable>
          ))}
        </View>

        <Pressable onPress={clearFilters}>
          <Text style={styles.clearFilters}>Clear search and filters</Text>
        </Pressable>

        {/* MENU LIST */}

        <View style={styles.listHeader}>
          <Text style={styles.sectionTitle}>Your Menu Items</Text>
          <Text style={styles.itemCount}>
            {filteredItems.length} found
          </Text>
        </View>

        {!isLoaded ? (
          <Text style={styles.emptyText}>Loading your menu...</Text>
        ) : filteredItems.length === 0 ? (
          <View style={styles.emptyCard}>
            <Text style={styles.emptyEmoji}>🍽️</Text>
            <Text style={styles.emptyTitle}>No dishes found</Text>
            <Text style={styles.emptyText}>
              Try changing your search or add a new menu item.
            </Text>
          </View>
        ) : (
          filteredItems.map(item => (
            <View key={item.id} style={styles.menuCard}>
              <View style={styles.dishEmoji}>
                <Text style={styles.dishEmojiText}>
                  {item.category === 'Starter'
                    ? '🥗'
                    : item.category === 'Main Course'
                    ? '🍲'
                    : '🍰'}
                </Text>
              </View>

              <View style={styles.dishInfo}>
                <Text style={styles.dishName}>{item.name}</Text>

                <Text style={styles.dishCategory}>
                  {item.category}
                </Text>

                <Text style={styles.dishDescription}>
                  {item.description}
                </Text>

                <Text style={styles.dishPrice}>
                  R{item.price.toFixed(2)}
                </Text>

                <View style={styles.actionContainer}>
                  <Pressable
                    style={styles.editButton}
                    onPress={() => openEditForm(item)}
                  >
                    <Text style={styles.editButtonText}>Edit</Text>
                  </Pressable>

                  <Pressable
                    style={styles.deleteButton}
                    onPress={() => deleteMenuItem(item.id)}
                  >
                    <Text style={styles.deleteButtonText}>Delete</Text>
                  </Pressable>
                </View>
              </View>
            </View>
          ))
        )}

        {/* FOOTER */}

        <View style={styles.footer}>
          <Text style={styles.footerEmoji}>❤️</Text>
          <Text style={styles.footerText}>
            Made with passion, served with love.
          </Text>
          <Text style={styles.footerCopyright}>
            Chef's Menu Manager
          </Text>
        </View>
      </ScrollView>

      {/* ADD AND EDIT MODAL */}

      <Modal
        visible={modalVisible}
        animationType="slide"
        transparent
        onRequestClose={() => {
          setModalVisible(false);
          resetForm();
        }}
      >
        <KeyboardAvoidingView
          style={styles.modalOverlay}
          behavior={Platform.OS === 'ios' ? 'padding' : undefined}
        >
          <View style={styles.modalCard}>
            <ScrollView
              keyboardShouldPersistTaps="handled"
              showsVerticalScrollIndicator={false}
            >
              <Text style={styles.modalTitle}>
                {editingId ? 'Edit Menu Item' : 'Add Menu Item'}
              </Text>

              <Text style={styles.modalSubtitle}>
                Enter the details of your dish below.
              </Text>

              <Text style={styles.fieldLabel}>Dish Name</Text>

              <TextInput
                style={styles.formInput}
                placeholder="Enter dish name"
                placeholderTextColor="#999999"
                value={dishName}
                onChangeText={setDishName}
              />

              <Text style={styles.fieldLabel}>Description</Text>

              <TextInput
                style={[styles.formInput, styles.descriptionInput]}
                placeholder="Describe your dish"
                placeholderTextColor="#999999"
                value={dishDescription}
                onChangeText={setDishDescription}
                multiline
              />

              <Text style={styles.fieldLabel}>Price (R)</Text>

              <TextInput
                style={styles.formInput}
                placeholder="Enter price"
                placeholderTextColor="#999999"
                value={dishPrice}
                onChangeText={setDishPrice}
                keyboardType="decimal-pad"
              />

              <Text style={styles.fieldLabel}>Course Category</Text>

              <View style={styles.categoryOptions}>
                {CATEGORIES.map(category => (
                  <Pressable
                    key={category}
                    onPress={() => setDishCategory(category)}
                    style={[
                      styles.categoryOption,
                      dishCategory === category &&
                        styles.selectedCategoryOption,
                    ]}
                  >
                    <Text
                      style={[
                        styles.categoryOptionText,
                        dishCategory === category &&
                          styles.selectedCategoryText,
                      ]}
                    >
                      {category}
                    </Text>
                  </Pressable>
                ))}
              </View>

              <Pressable
                style={styles.saveButton}
                onPress={saveMenuItem}
              >
                <Text style={styles.saveButtonText}>
                  {editingId ? 'Update Menu Item' : 'Save Menu Item'}
                </Text>
              </Pressable>

              <Pressable
                style={styles.cancelButton}
                onPress={() => {
                  setModalVisible(false);
                  resetForm();
                }}
              >
                <Text style={styles.cancelButtonText}>Cancel</Text>
              </Pressable>
            </ScrollView>
          </View>
        </KeyboardAvoidingView>
      </Modal>
    </SafeAreaView>
  );
}

// --------------------------------------------------
// 14. REUSABLE STATISTICS CARD
// --------------------------------------------------

function StatCard({
  emoji,
  title,
  value,
  fullWidth = false,
}: {
  emoji: string;
  title: string;
  value: string;
  fullWidth?: boolean;
}) {
  return (
    <View
      style={[
        styles.statCard,
        fullWidth && styles.fullWidthCard,
      ]}
    >
      <Text style={styles.statEmoji}>{emoji}</Text>
      <Text style={styles.statValue}>{value}</Text>
      <Text style={styles.statTitle}>{title}</Text>
    </View>
  );
}

// --------------------------------------------------
// 15. APPLICATION STYLING
// --------------------------------------------------

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#F8F5EF',
  },
  scrollContent: {
    padding: 20,
    paddingBottom: 40,
  },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginBottom: 25,
  },
  greeting: {
    fontSize: 11,
    letterSpacing: 2,
    color: '#A47C52',
    fontWeight: '700',
  },
  title: {
    fontSize: 29,
    fontWeight: 'bold',
    color: '#263D32',
    marginTop: 5,
  },
  subtitle: {
    fontSize: 15,
    color: '#7B817A',
  },
  profileIcon: {
    width: 55,
    height: 55,
    borderRadius: 28,
    backgroundColor: '#E9E2D5',
    justifyContent: 'center',
    alignItems: 'center',
  },
  profileText: {
    fontSize: 28,
  },
  banner: {
    backgroundColor: '#263D32',
    borderRadius: 20,
    padding: 23,
    marginBottom: 28,
    overflow: 'hidden',
  },
  bannerLabel: {
    color: '#E8B77C',
    fontSize: 10,
    letterSpacing: 2,
    fontWeight: 'bold',
  },
  bannerTitle: {
    color: '#FFFFFF',
    fontSize: 23,
    fontWeight: 'bold',
    marginTop: 13,
    maxWidth: '85%',
  },
  bannerDescription: {
    color: '#E2E8E1',
    fontSize: 13,
    lineHeight: 20,
    marginTop: 9,
    maxWidth: '85%',
  },
  bannerEmoji: {
    fontSize: 48,
    position: 'absolute',
    right: 13,
    bottom: 12,
  },
  sectionHeader: {
    marginBottom: 15,
  },
  sectionTitle: {
    color: '#263D32',
    fontSize: 20,
    fontWeight: 'bold',
  },
  sectionSubtitle: {
    color: '#92938D',
    fontSize: 12,
    marginTop: 4,
  },
  statsGrid: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    justifyContent: 'space-between',
    marginBottom: 28,
  },
  statCard: {
    width: '48%',
    backgroundColor: '#FFFFFF',
    borderRadius: 16,
    padding: 17,
    marginBottom: 12,
    borderWidth: 1,
    borderColor: '#EEE9DF',
  },
  fullWidthCard: {
    width: '100%',
  },
  statEmoji: {
    fontSize: 23,
  },
  statValue: {
    fontSize: 25,
    fontWeight: 'bold',
    color: '#263D32',
    marginTop: 10,
  },
  statTitle: {
    fontSize: 12,
    color: '#85877F',
    marginTop: 3,
  },
  menuHeader: {
    marginBottom: 15,
  },
  addButton: {
    backgroundColor: '#263D32',
    padding: 16,
    borderRadius: 13,
    alignItems: 'center',
    marginBottom: 20,
  },
  addButtonText: {
    color: '#FFFFFF',
    fontSize: 14,
    fontWeight: 'bold',
  },
  fieldLabel: {
    color: '#263D32',
    fontSize: 13,
    fontWeight: 'bold',
    marginBottom: 8,
    marginTop: 12,
  },
  searchInput: {
    backgroundColor: '#FFFFFF',
    borderRadius: 12,
    borderWidth: 1,
    borderColor: '#E8E1D6',
    padding: 14,
    color: '#263D32',
    fontSize: 14,
  },
  filterContainer: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    gap: 7,
  },
  filterButton: {
    backgroundColor: '#FFFFFF',
    borderRadius: 20,
    paddingVertical: 9,
    paddingHorizontal: 12,
    borderWidth: 1,
    borderColor: '#E8E1D6',
  },
  activeFilterButton: {
    backgroundColor: '#263D32',
    borderColor: '#263D32',
  },
  filterText: {
    fontSize: 11,
    color: '#59665C',
    fontWeight: '600',
  },
  activeFilterText: {
    color: '#FFFFFF',
  },
  clearFilters: {
    color: '#A47C52',
    fontSize: 12,
    fontWeight: 'bold',
    marginTop: 12,
    marginBottom: 25,
  },
  listHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginBottom: 15,
  },
  itemCount: {
    color: '#92938D',
    fontSize: 12,
  },
  menuCard: {
    backgroundColor: '#FFFFFF',
    borderRadius: 15,
    padding: 14,
    marginBottom: 12,
    flexDirection: 'row',
    alignItems: 'flex-start',
    borderWidth: 1,
    borderColor: '#EEE9DF',
  },
  dishEmoji: {
    width: 48,
    height: 48,
    borderRadius: 13,
    backgroundColor: '#F3EFE7',
    justifyContent: 'center',
    alignItems: 'center',
  },
  dishEmojiText: {
    fontSize: 25,
  },
  dishInfo: {
    flex: 1,
    marginLeft: 12,
  },
  dishName: {
    color: '#263D32',
    fontSize: 14,
    fontWeight: 'bold',
  },
  dishCategory: {
    color: '#A47C52',
    fontSize: 11,
    marginTop: 4,
    fontWeight: '600',
  },
  dishDescription: {
    color: '#85877F',
    fontSize: 11,
    lineHeight: 17,
    marginTop: 5,
  },
  dishPrice: {
    color: '#263D32',
    fontSize: 14,
    fontWeight: 'bold',
    marginTop: 7,
  },
  actionContainer: {
    flexDirection: 'row',
    gap: 9,
    marginTop: 12,
  },
  editButton: {
    backgroundColor: '#E8EFE9',
    borderRadius: 8,
    paddingVertical: 8,
    paddingHorizontal: 17,
  },
  editButtonText: {
    color: '#263D32',
    fontSize: 11,
    fontWeight: 'bold',
  },
  deleteButton: {
    backgroundColor: '#FCEAEA',
    borderRadius: 8,
    paddingVertical: 8,
    paddingHorizontal: 17,
  },
  deleteButtonText: {
    color: '#B54747',
    fontSize: 11,
    fontWeight: 'bold',
  },
  emptyCard: {
    backgroundColor: '#FFFFFF',
    borderRadius: 15,
    padding: 25,
    alignItems: 'center',
  },
  emptyEmoji: {
    fontSize: 35,
  },
  emptyTitle: {
    color: '#263D32',
    fontSize: 16,
    fontWeight: 'bold',
    marginTop: 12,
  },
  emptyText: {
    color: '#85877F',
    fontSize: 12,
    textAlign: 'center',
    marginTop: 8,
  },
  footer: {
    alignItems: 'center',
    marginTop: 25,
    paddingTop: 20,
    borderTopWidth: 1,
    borderTopColor: '#E9E2D8',
  },
  footerEmoji: {
    fontSize: 18,
  },
  footerText: {
    color: '#6F786F',
    fontSize: 12,
    marginTop: 7,
  },
  footerCopyright: {
    color: '#A1A198',
    fontSize: 10,
    marginTop: 7,
  },
  modalOverlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.55)',
    justifyContent: 'center',
    padding: 18,
  },
  modalCard: {
    backgroundColor: '#F8F5EF',
    borderRadius: 20,
    padding: 22,
    maxHeight: '90%',
  },
  modalTitle: {
    color: '#263D32',
    fontSize: 23,
    fontWeight: 'bold',
  },
  modalSubtitle: {
    color: '#85877F',
    fontSize: 12,
    marginTop: 5,
    marginBottom: 10,
  },
  formInput: {
    backgroundColor: '#FFFFFF',
    borderRadius: 10,
    borderWidth: 1,
    borderColor: '#E8E1D6',
    padding: 13,
    fontSize: 14,
    color: '#263D32',
  },
  descriptionInput: {
    height: 85,
    textAlignVertical: 'top',
  },
  categoryOptions: {
    gap: 8,
  },
  categoryOption: {
    backgroundColor: '#FFFFFF',
    padding: 12,
    borderRadius: 10,
    borderWidth: 1,
    borderColor: '#E8E1D6',
  },
  selectedCategoryOption: {
    backgroundColor: '#263D32',
    borderColor: '#263D32',
  },
  categoryOptionText: {
    color: '#59665C',
    fontSize: 13,
    textAlign: 'center',
    fontWeight: '600',
  },
  selectedCategoryText: {
    color: '#FFFFFF',
  },
  saveButton: {
    backgroundColor: '#263D32',
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
    marginTop: 24,
  },
  saveButtonText: {
    color: '#FFFFFF',
    fontSize: 14,
    fontWeight: 'bold',
  },
  cancelButton: {
    backgroundColor: '#E9E2D5',
    padding: 14,
    borderRadius: 12,
    alignItems: 'center',
    marginTop: 10,
    marginBottom: 5,
  },
  cancelButtonText: {
    color: '#263D32',
    fontSize: 13,
    fontWeight: 'bold',
  },
});
