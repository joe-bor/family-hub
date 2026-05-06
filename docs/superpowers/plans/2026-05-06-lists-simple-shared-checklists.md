# Lists Simple Shared Checklists Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the first real `Lists` module as a family-owned hub of separate lists with kind-gated categories, completed-item visibility controls, and persisted create / edit / complete / clear behavior across backend and frontend.

**Architecture:** Implement the backend list contract first in `backend/family-hub-api/`, including seeded family-scoped categories and preferences, then wire the frontend `Lists` hub and detail views in `frontend/` against that contract. Backend owns persistence, seed provisioning, family-scoped validation, and the `/api/lists` surface; frontend owns grouped-vs-flat presentation, completed-visibility resolution, mobile create flows, and optimistic list/item UX.

**Tech Stack:** Spring Boot 4, Java 21, JPA/Hibernate, Flyway, Maven, React 19, TypeScript, TanStack Query, React Hook Form, Zod, Vitest, MSW, Playwright.

**Spec:** `docs/superpowers/specs/2026-05-06-lists-simple-shared-checklists-design.md`

---

## Execution Order

1. Backend schema, seed provisioning, and `/api/lists` contract
2. Backend verification and published BE release
3. Frontend types, mocks, hooks, and validation against the stable contract
4. Frontend hub and detail UI
5. Frontend live verification against the released backend

Per root shipping rules, live FE verification that depends on real `/api/lists` behavior must wait for a published backend release. Mock-backed FE work can begin earlier, but E2E should use the released backend image, not unreleased backend `main`.

## Execution Contract

- `Lists` is family-scoped shared state, not per-user personal state.
- Families can create multiple lists, and each list must have a required `name` and required `kind`.
- `grocery` and `to-do` lists get seeded read-only family category sets; `general` lists do not use categories.
- Categories are optional per item, and category-aware lists must support grouped and flat display modes.
- The module must not add assignees, due dates, reminders, recurrence, or other `Chores` workflow semantics.
- Families get a module-wide completed-items default, and individual lists can override it.
- Whenever completed items are shown, they must stay checked, muted, and struck through.
- Each list needs a manual `Remove all completed` action, and completed items remain list data even when hidden.
- MVP does not include custom category management, list rename/delete flows, or list templates.

## File Structure

### Root docs / orchestration

- Create: `docs/superpowers/plans/2026-05-06-lists-simple-shared-checklists.md`
- Modify later during orchestration only: `docs/product/backlog/module-foundations/lists-simple-shared-checklists.md`

### Backend (`backend/family-hub-api/`)

Expected create / modify set:

- Create: `src/main/resources/db/migration/V12__create_lists_tables.sql`
- Create: `src/main/java/com/familyhub/demo/model/ListKind.java`
- Create: `src/main/java/com/familyhub/demo/model/ListCategoryDisplayMode.java`
- Create: `src/main/java/com/familyhub/demo/model/SharedList.java`
- Create: `src/main/java/com/familyhub/demo/model/SharedListItem.java`
- Create: `src/main/java/com/familyhub/demo/model/ListCategory.java`
- Create: `src/main/java/com/familyhub/demo/model/ListPreferences.java`
- Create: `src/main/java/com/familyhub/demo/dto/CreateListRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/UpdateListRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/CreateListItemRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/UpdateListItemRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/ListSummaryResponse.java`
- Create: `src/main/java/com/familyhub/demo/dto/ListDetailResponse.java`
- Create: `src/main/java/com/familyhub/demo/dto/ListItemResponse.java`
- Create: `src/main/java/com/familyhub/demo/dto/ListCategoryResponse.java`
- Create: `src/main/java/com/familyhub/demo/dto/ListPreferencesResponse.java`
- Create: `src/main/java/com/familyhub/demo/dto/UpdateListPreferencesRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/ClearCompletedResponse.java`
- Create: `src/main/java/com/familyhub/demo/mapper/ListMapper.java`
- Create: `src/main/java/com/familyhub/demo/repository/SharedListRepository.java`
- Create: `src/main/java/com/familyhub/demo/repository/ListCategoryRepository.java`
- Create: `src/main/java/com/familyhub/demo/repository/ListPreferencesRepository.java`
- Create: `src/main/java/com/familyhub/demo/service/ListSeedService.java`
- Create: `src/main/java/com/familyhub/demo/service/ListService.java`
- Create: `src/main/java/com/familyhub/demo/controller/ListController.java`
- Modify: `src/main/java/com/familyhub/demo/service/AuthService.java`
- Modify: `src/test/java/com/familyhub/demo/TestDataFactory.java`
- Modify: `src/test/java/com/familyhub/demo/service/AuthServiceTest.java`
- Create: `src/test/java/com/familyhub/demo/service/ListServiceTest.java`
- Create: `src/test/java/com/familyhub/demo/controller/ListControllerTest.java`
- Create: `src/test/java/com/familyhub/demo/integration/ListIntegrationTest.java`

### Frontend (`frontend/`)

Expected create / modify set:

- Create: `src/lib/types/lists.ts`
- Modify: `src/lib/types/index.ts`
- Create: `src/api/services/lists.service.ts`
- Modify: `src/api/services/index.ts`
- Create: `src/api/hooks/use-lists.ts`
- Modify: `src/api/hooks/index.ts`
- Modify: `src/api/index.ts`
- Create: `src/lib/validations/lists.ts`
- Create: `src/lib/validations/lists.test.ts`
- Modify: `src/lib/validations/index.ts`
- Modify: `src/test/mocks/handlers.ts`
- Modify: `src/test/mocks/server.ts`
- Delete: `src/stores/lists-store.ts`
- Modify: `src/stores/index.ts`
- Create: `src/components/ui/mobile-sheet.tsx`
- Modify: `src/components/calendar/components/mobile-event-sheet.tsx`
- Create: `src/components/lists/list-create-sheet.tsx`
- Create: `src/components/lists/list-item-sheet.tsx`
- Create: `src/components/lists/list-card.tsx`
- Create: `src/components/lists/list-detail-view.tsx`
- Create: `src/components/lists/list-item-row.tsx`
- Create: `src/components/lists/build-list-sections.ts`
- Create: `src/components/lists/build-list-sections.test.ts`
- Modify: `src/components/lists-view.tsx`
- Create: `src/api/hooks/use-lists.test.tsx`
- Create: `src/components/lists-view.test.tsx`
- Create: `e2e/mobile-lists.spec.ts`

This structure keeps list data contracts in `@/api`, presentation logic in `src/components/lists/`, and the list-grouping rules isolated in a small helper instead of bloating `src/components/lists-view.tsx`.

## Task 1: Seed Backend Lists Foundations

**Files:**
- Create: `backend/family-hub-api/src/main/resources/db/migration/V12__create_lists_tables.sql`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/ListKind.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/ListCategoryDisplayMode.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/SharedList.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/SharedListItem.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/ListCategory.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/ListPreferences.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/repository/ListCategoryRepository.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/repository/ListPreferencesRepository.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/service/ListSeedService.java`
- Modify: `backend/family-hub-api/src/main/java/com/familyhub/demo/service/AuthService.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/service/AuthServiceTest.java`

- [ ] **Step 1: Write the failing registration test that proves new families get seeded list defaults**

```java
@Mock
private ListSeedService listSeedService;

@Test
void register_success_seedsListsModuleDefaults() {
    RegisterRequest request = createRegisterRequest();
    when(familyRepository.existsByUsername(request.username())).thenReturn(false);
    when(passwordEncoder.encode(anyString())).thenReturn("encoded-password");
    when(familyRepository.saveAndFlush(any(Family.class))).thenReturn(family);
    when(jwtService.generateToken(family)).thenReturn("jwt-token");

    authService.register(request);

    verify(listSeedService).seedDefaultsForFamily(family);
}
```

- [ ] **Step 2: Run the auth-service test to verify it fails because the seed service does not exist yet**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=AuthServiceTest test
```

Expected: FAIL with compilation errors for `ListSeedService` / missing constructor injection.

- [ ] **Step 3: Add the migration, enums, entities, repositories, and seed service**

```sql
ALTER TABLE family_member
    ADD CONSTRAINT uk_family_member_family_id_id UNIQUE (family_id, id);

CREATE TABLE list_preferences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL UNIQUE,
    show_completed_by_default BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP(6) DEFAULT now(),
    updated_at TIMESTAMP(6) DEFAULT now(),
    CONSTRAINT fk_list_preferences_family
        FOREIGN KEY (family_id) REFERENCES family(id) ON DELETE CASCADE
);

CREATE TABLE list_category (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL,
    kind VARCHAR(20) NOT NULL,
    name VARCHAR(100) NOT NULL,
    seeded BOOLEAN NOT NULL DEFAULT TRUE,
    sort_order INTEGER NOT NULL,
    created_at TIMESTAMP(6) DEFAULT now(),
    updated_at TIMESTAMP(6) DEFAULT now(),
    CONSTRAINT fk_list_category_family
        FOREIGN KEY (family_id) REFERENCES family(id) ON DELETE CASCADE,
    CONSTRAINT uk_list_category_family_kind_name UNIQUE (family_id, kind, name)
);

CREATE TABLE shared_list (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL,
    name VARCHAR(100) NOT NULL,
    kind VARCHAR(20) NOT NULL,
    category_display_mode VARCHAR(20) NOT NULL,
    show_completed_override BOOLEAN,
    created_at TIMESTAMP(6) DEFAULT now(),
    updated_at TIMESTAMP(6) DEFAULT now(),
    CONSTRAINT fk_shared_list_family
        FOREIGN KEY (family_id) REFERENCES family(id) ON DELETE CASCADE
);

CREATE TABLE shared_list_item (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    list_id UUID NOT NULL,
    category_id UUID,
    text VARCHAR(100) NOT NULL,
    completed BOOLEAN NOT NULL DEFAULT FALSE,
    completed_at TIMESTAMP(6),
    created_at TIMESTAMP(6) DEFAULT now(),
    updated_at TIMESTAMP(6) DEFAULT now(),
    CONSTRAINT fk_shared_list_item_list
        FOREIGN KEY (list_id) REFERENCES shared_list(id) ON DELETE CASCADE,
    CONSTRAINT fk_shared_list_item_category
        FOREIGN KEY (category_id) REFERENCES list_category(id) ON DELETE SET NULL
);

INSERT INTO list_preferences (family_id, show_completed_by_default)
SELECT id, TRUE
FROM family
ON CONFLICT (family_id) DO NOTHING;

INSERT INTO list_category (family_id, kind, name, seeded, sort_order)
SELECT f.id, seed.kind, seed.name, TRUE, seed.sort_order
FROM family f
CROSS JOIN (
    VALUES
        ('grocery', 'Produce', 0),
        ('grocery', 'Dairy', 1),
        ('grocery', 'Pantry', 2),
        ('grocery', 'Frozen', 3),
        ('grocery', 'Household', 4),
        ('to-do', 'Urgent', 0),
        ('to-do', 'Soon', 1),
        ('to-do', 'Later', 2)
) AS seed(kind, name, sort_order)
ON CONFLICT (family_id, kind, name) DO NOTHING;
```

```java
public enum ListKind {
    GROCERY("grocery"),
    TODO("to-do"),
    GENERAL("general");

    private final String wireValue;

    ListKind(String wireValue) {
        this.wireValue = wireValue;
    }

    @JsonValue
    public String wireValue() {
        return wireValue;
    }
}
```

```java
public enum ListCategoryDisplayMode {
    GROUPED("grouped"),
    FLAT("flat");

    private final String wireValue;

    ListCategoryDisplayMode(String wireValue) {
        this.wireValue = wireValue;
    }

    @JsonValue
    public String wireValue() {
        return wireValue;
    }
}
```

```java
@Entity
@Table(name = "shared_list")
@Getter
@Setter
public class SharedList {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @ManyToOne
    @JoinColumn(name = "family_id", nullable = false)
    private Family family;

    @Column(nullable = false, length = 100)
    private String name;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private ListKind kind;

    @Enumerated(EnumType.STRING)
    @Column(name = "category_display_mode", nullable = false, length = 20)
    private ListCategoryDisplayMode categoryDisplayMode;

    @Column(name = "show_completed_override")
    private Boolean showCompletedOverride;

    @OneToMany(mappedBy = "list", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("createdAt ASC")
    private List<SharedListItem> items = new ArrayList<>();

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}
```

```java
@Service
@RequiredArgsConstructor
public class ListSeedService {
    private final ListPreferencesRepository listPreferencesRepository;
    private final ListCategoryRepository listCategoryRepository;

    @Transactional
    public void seedDefaultsForFamily(Family family) {
        listPreferencesRepository.findByFamily(family)
                .orElseGet(() -> {
                    ListPreferences preferences = new ListPreferences();
                    preferences.setFamily(family);
                    preferences.setShowCompletedByDefault(true);
                    return listPreferencesRepository.save(preferences);
                });

        seedKind(family, ListKind.GROCERY, List.of("Produce", "Dairy", "Pantry", "Frozen", "Household"));
        seedKind(family, ListKind.TODO, List.of("Urgent", "Soon", "Later"));
    }

    private void seedKind(Family family, ListKind kind, List<String> names) {
        if (listCategoryRepository.countByFamilyAndKind(family, kind) > 0) {
            return;
        }

        IntStream.range(0, names.size()).forEach(index -> {
            ListCategory category = new ListCategory();
            category.setFamily(family);
            category.setKind(kind);
            category.setName(names.get(index));
            category.setSeeded(true);
            category.setSortOrder(index);
            listCategoryRepository.save(category);
        });
    }
}
```

```java
@Transactional
public AuthResponse register(RegisterRequest registerRequest) {
    // existing save logic
    Family saved = familyRepository.saveAndFlush(family);
    listSeedService.seedDefaultsForFamily(saved);
    String token = jwtService.generateToken(saved);
    return new AuthResponse(token, FamilyMapper.toDto(saved));
}
```

- [ ] **Step 4: Run the auth-service test again to verify seeding is wired**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=AuthServiceTest test
```

Expected: PASS, including the new `register_success_seedsListsModuleDefaults` test.

- [ ] **Step 5: Commit**

```bash
cd backend/family-hub-api
git add src/main/resources/db/migration/V12__create_lists_tables.sql \
  src/main/java/com/familyhub/demo/model/ListKind.java \
  src/main/java/com/familyhub/demo/model/ListCategoryDisplayMode.java \
  src/main/java/com/familyhub/demo/model/SharedList.java \
  src/main/java/com/familyhub/demo/model/SharedListItem.java \
  src/main/java/com/familyhub/demo/model/ListCategory.java \
  src/main/java/com/familyhub/demo/model/ListPreferences.java \
  src/main/java/com/familyhub/demo/repository/ListCategoryRepository.java \
  src/main/java/com/familyhub/demo/repository/ListPreferencesRepository.java \
  src/main/java/com/familyhub/demo/service/ListSeedService.java \
  src/main/java/com/familyhub/demo/service/AuthService.java \
  src/test/java/com/familyhub/demo/service/AuthServiceTest.java
git commit -m "feat(lists): seed backend list defaults"
```

## Task 2: Implement Backend Lists API And Service Rules

**Files:**
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/CreateListRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/UpdateListRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/CreateListItemRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/UpdateListItemRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/ListSummaryResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/ListDetailResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/ListItemResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/ListCategoryResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/ListPreferencesResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/UpdateListPreferencesRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/ClearCompletedResponse.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/mapper/ListMapper.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/repository/SharedListRepository.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/service/ListService.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/controller/ListController.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/TestDataFactory.java`
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/service/ListServiceTest.java`
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/controller/ListControllerTest.java`

- [ ] **Step 1: Write the failing controller tests for the full `/api/lists` surface**

```java
@WebMvcTest(ListController.class)
@Import({SecurityConfig.class, JwtAuthenticationFilter.class, JwtAuthenticationEntryPoint.class})
@ActiveProfiles("test")
class ListControllerTest {

    @Autowired
    MockMvc mockMvc;

    @MockitoBean
    JwtService jwtService;

    @MockitoBean
    FamilyService familyService;

    @MockitoBean
    ListService listService;

    @Test
    @WithMockFamily
    void getLists_returns200() throws Exception {
        given(listService.getLists(any(Family.class)))
                .willReturn(List.of(sampleListSummaryResponse()));

        mockMvc.perform(get("/api/lists"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data[0].name").value("Trader Joe's Run"))
                .andExpect(jsonPath("$.data[0].kind").value("grocery"));
    }

    @Test
    @WithMockFamily
    void createList_returns201() throws Exception {
        given(listService.createList(any(), any(Family.class)))
                .willReturn(sampleListDetailResponse());

        mockMvc.perform(post("/api/lists")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {
                                    "name": "Trader Joe's Run",
                                    "kind": "grocery"
                                }
                                """))
                .andExpect(status().isCreated())
                .andExpect(header().string("Location", "/api/lists/" + LIST_ID))
                .andExpect(jsonPath("$.message").value("List created successfully"));
    }

    @Test
    @WithMockFamily
    void patchList_updatesDisplayModeAndOverride() throws Exception {
        given(listService.updateList(eq(LIST_ID), any(), any(Family.class)))
                .willReturn(sampleListDetailResponse());

        mockMvc.perform(patch("/api/lists/{id}", LIST_ID)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {
                                    "categoryDisplayMode": "flat",
                                    "showCompletedOverride": false
                                }
                                """))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.message").value("List updated successfully"));
    }

    @Test
    @WithMockFamily
    void clearCompleted_returnsRemovedCount() throws Exception {
        given(listService.clearCompleted(eq(LIST_ID), any(Family.class)))
                .willReturn(new ClearCompletedResponse(2));

        mockMvc.perform(post("/api/lists/{id}/clear-completed", LIST_ID))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.removedCount").value(2));
    }

    @Test
    @WithMockFamily
    void getPreferences_returns200() throws Exception {
        given(listService.getPreferences(any(Family.class)))
                .willReturn(new ListPreferencesResponse(true));

        mockMvc.perform(get("/api/lists/preferences"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.showCompletedByDefault").value(true));
    }
}
```

- [ ] **Step 2: Write the failing service tests for category rules and bulk clear behavior**

```java
@ExtendWith(MockitoExtension.class)
class ListServiceTest {

    @Mock
    private SharedListRepository sharedListRepository;

    @Mock
    private ListCategoryRepository listCategoryRepository;

    @Mock
    private ListPreferencesRepository listPreferencesRepository;

    @InjectMocks
    private ListService listService;

    private Family family;
    private SharedList groceryList;
    private ListCategory produceCategory;

    @BeforeEach
    void setUp() {
        family = createFamily();
        groceryList = createGroceryList(family);
        produceCategory = createListCategory(family, ListKind.GROCERY, "Produce", 0);
    }

    @Test
    void createItem_generalListWithCategory_throwsBadRequest() {
        SharedList generalList = createGeneralList(family);
        when(sharedListRepository.findDetailByFamilyAndId(family, LIST_ID)).thenReturn(Optional.of(generalList));

        assertThatThrownBy(() -> listService.createItem(
                LIST_ID,
                new CreateListItemRequest("Paper towels", produceCategory.getId()),
                family
        )).isInstanceOf(BadRequestException.class);
    }

    @Test
    void updateItem_categoryFromWrongKind_throwsBadRequest() {
        ListCategory todoCategory = createListCategory(family, ListKind.TODO, "Urgent", 0);
        when(sharedListRepository.findDetailByFamilyAndId(family, LIST_ID)).thenReturn(Optional.of(groceryList));
        when(listCategoryRepository.findByFamilyAndId(family, todoCategory.getId())).thenReturn(Optional.of(todoCategory));

        assertThatThrownBy(() -> listService.updateItem(
                LIST_ID,
                LIST_ITEM_ID,
                new UpdateListItemRequest("Bananas", false, todoCategory.getId()),
                family
        )).isInstanceOf(BadRequestException.class);
    }

    @Test
    void clearCompleted_removesCompletedItemsAndReturnsCount() {
        when(sharedListRepository.findDetailByFamilyAndId(family, LIST_ID)).thenReturn(Optional.of(createListWithCompletedItems(family)));
        when(sharedListRepository.save(any(SharedList.class))).thenAnswer(invocation -> invocation.getArgument(0));

        ClearCompletedResponse response = listService.clearCompleted(LIST_ID, family);

        assertThat(response.removedCount()).isEqualTo(2);
    }
}
```

- [ ] **Step 3: Add the DTOs, mapper, repository queries, service, and controller**

```java
public static final UUID LIST_ID = UUID.fromString("00000000-0000-0000-0000-000000000005");
public static final UUID LIST_ITEM_ID = UUID.fromString("00000000-0000-0000-0000-000000000006");
public static final UUID LIST_CATEGORY_ID = UUID.fromString("00000000-0000-0000-0000-000000000007");

public static SharedList createGroceryList(Family family) {
    SharedList list = new SharedList();
    list.setId(LIST_ID);
    list.setFamily(family);
    list.setName("Trader Joe's Run");
    list.setKind(ListKind.GROCERY);
    list.setCategoryDisplayMode(ListCategoryDisplayMode.GROUPED);
    list.setShowCompletedOverride(null);
    list.setItems(new ArrayList<>());
    list.setCreatedAt(LocalDateTime.of(2026, 5, 6, 9, 0));
    list.setUpdatedAt(LocalDateTime.of(2026, 5, 6, 9, 0));
    return list;
}

public static SharedList createGeneralList(Family family) {
    SharedList list = createGroceryList(family);
    list.setKind(ListKind.GENERAL);
    list.setCategoryDisplayMode(ListCategoryDisplayMode.FLAT);
    list.setName("Movies to Watch");
    return list;
}

public static ListCategory createListCategory(Family family, ListKind kind, String name, int sortOrder) {
    ListCategory category = new ListCategory();
    category.setId(LIST_CATEGORY_ID);
    category.setFamily(family);
    category.setKind(kind);
    category.setName(name);
    category.setSeeded(true);
    category.setSortOrder(sortOrder);
    return category;
}

public static SharedList createListWithCompletedItems(Family family) {
    SharedList list = createGroceryList(family);

    SharedListItem bananas = new SharedListItem();
    bananas.setId(LIST_ITEM_ID);
    bananas.setList(list);
    bananas.setText("Bananas");
    bananas.setCompleted(true);
    bananas.setCompletedAt(LocalDateTime.of(2026, 5, 6, 10, 0));

    SharedListItem spinach = new SharedListItem();
    spinach.setId(UUID.fromString("00000000-0000-0000-0000-000000000008"));
    spinach.setList(list);
    spinach.setText("Spinach");
    spinach.setCompleted(true);
    spinach.setCompletedAt(LocalDateTime.of(2026, 5, 6, 10, 5));

    SharedListItem yogurt = new SharedListItem();
    yogurt.setId(UUID.fromString("00000000-0000-0000-0000-000000000009"));
    yogurt.setList(list);
    yogurt.setText("Greek yogurt");
    yogurt.setCompleted(false);
    yogurt.setCompletedAt(null);

    list.setItems(new ArrayList<>(List.of(bananas, spinach, yogurt)));
    return list;
}

public static ListSummaryResponse sampleListSummaryResponse() {
    return new ListSummaryResponse(
            LIST_ID,
            "Trader Joe's Run",
            "grocery",
            3,
            1
    );
}
```

```java
public record CreateListRequest(
        @NotBlank
        @Size(max = 100, message = "List name must be 100 characters or less")
        String name,

        @NotNull
        ListKind kind
) {}

public record UpdateListRequest(
        @NotNull
        ListCategoryDisplayMode categoryDisplayMode,

        Boolean showCompletedOverride
) {}

public record CreateListItemRequest(
        @NotBlank
        @Size(max = 100, message = "Item text must be 100 characters or less")
        String text,

        UUID categoryId
) {}

public record UpdateListItemRequest(
        @NotBlank
        @Size(max = 100, message = "Item text must be 100 characters or less")
        String text,

        @NotNull
        Boolean completed,

        UUID categoryId
) {}
```

```java
@Query("""
        select distinct l from SharedList l
        left join fetch l.items item
        left join fetch item.category
        where l.family = :family
        order by l.createdAt desc
        """)
List<SharedList> findByFamilyWithItems(@Param("family") Family family);

@Query("""
        select distinct l from SharedList l
        left join fetch l.items item
        left join fetch item.category
        where l.family = :family
          and l.id = :id
        """)
Optional<SharedList> findDetailByFamilyAndId(@Param("family") Family family, @Param("id") UUID id);

long countByFamilyAndKind(Family family, ListKind kind);
Optional<ListCategory> findByFamilyAndId(Family family, UUID id);
List<ListCategory> findByFamilyAndKindOrderBySortOrderAsc(Family family, ListKind kind);
Optional<ListPreferences> findByFamily(Family family);
```

```java
public final class ListMapper {
    private ListMapper() {
    }

    public static ListSummaryResponse toSummaryDto(SharedList list) {
        int completedItems = (int) list.getItems().stream().filter(SharedListItem::isCompleted).count();
        return new ListSummaryResponse(
                list.getId(),
                list.getName(),
                list.getKind().wireValue(),
                list.getItems().size(),
                completedItems
        );
    }

    public static ListCategoryResponse toCategoryDto(ListCategory category) {
        return new ListCategoryResponse(
                category.getId(),
                category.getKind().wireValue(),
                category.getName(),
                category.isSeeded(),
                category.getSortOrder()
        );
    }

    public static ListItemResponse toItemDto(SharedListItem item) {
        return new ListItemResponse(
                item.getId(),
                item.getText(),
                item.isCompleted(),
                item.getCompletedAt(),
                item.getCategory() != null ? item.getCategory().getId() : null,
                item.getCreatedAt(),
                item.getUpdatedAt()
        );
    }

    public static ListDetailResponse toDetailDto(SharedList list, List<ListCategoryResponse> categories) {
        return new ListDetailResponse(
                list.getId(),
                list.getName(),
                list.getKind().wireValue(),
                list.getCategoryDisplayMode().wireValue(),
                list.getShowCompletedOverride(),
                categories,
                list.getItems().stream().map(ListMapper::toItemDto).toList(),
                list.getCreatedAt(),
                list.getUpdatedAt()
        );
    }
}

```

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ListService {
    private final SharedListRepository sharedListRepository;
    private final ListCategoryRepository listCategoryRepository;
    private final ListPreferencesRepository listPreferencesRepository;

    public List<ListSummaryResponse> getLists(Family family) {
        return sharedListRepository.findByFamilyWithItems(family)
                .stream()
                .map(ListMapper::toSummaryDto)
                .toList();
    }

    @Transactional
    public ListDetailResponse createList(CreateListRequest request, Family family) {
        SharedList list = new SharedList();
        list.setFamily(family);
        list.setName(request.name().trim());
        list.setKind(request.kind());
        list.setCategoryDisplayMode(request.kind() == ListKind.GENERAL
                ? ListCategoryDisplayMode.FLAT
                : ListCategoryDisplayMode.GROUPED);
        list.setShowCompletedOverride(null);
        return mapDetail(sharedListRepository.save(list));
    }

    public ListDetailResponse getList(UUID id, Family family) {
        return mapDetail(getListOrThrow(id, family));
    }

    @Transactional
    public ListDetailResponse updateList(UUID id, UpdateListRequest request, Family family) {
        SharedList list = getListOrThrow(id, family);
        if (list.getKind() == ListKind.GENERAL
                && request.categoryDisplayMode() == ListCategoryDisplayMode.GROUPED) {
            throw new BadRequestException("General lists cannot use grouped category mode.");
        }
        list.setCategoryDisplayMode(request.categoryDisplayMode());
        list.setShowCompletedOverride(request.showCompletedOverride());
        return mapDetail(sharedListRepository.save(list));
    }

    @Transactional
    public ListItemResponse createItem(UUID listId, CreateListItemRequest request, Family family) {
        SharedList list = getListOrThrow(listId, family);
        ListCategory category = resolveCategory(list, request.categoryId());

        SharedListItem item = new SharedListItem();
        item.setList(list);
        item.setText(request.text().trim());
        item.setCompleted(false);
        item.setCompletedAt(null);
        item.setCategory(category);
        list.getItems().add(item);

        SharedList saved = sharedListRepository.save(list);
        return ListMapper.toItemDto(saved.getItems().getLast());
    }

    @Transactional
    public ListItemResponse updateItem(UUID listId, UUID itemId, UpdateListItemRequest request, Family family) {
        SharedList list = getListOrThrow(listId, family);
        SharedListItem item = list.getItems().stream()
                .filter(candidate -> candidate.getId().equals(itemId))
                .findFirst()
                .orElseThrow(() -> new ResourceNotFoundException("List Item", itemId));

        item.setText(request.text().trim());
        boolean completed = Boolean.TRUE.equals(request.completed());
        item.setCompleted(completed);
        item.setCompletedAt(completed ? Optional.ofNullable(item.getCompletedAt()).orElse(LocalDateTime.now()) : null);
        item.setCategory(resolveCategory(list, request.categoryId()));

        sharedListRepository.save(list);
        return ListMapper.toItemDto(item);
    }

    @Transactional
    public void deleteItem(UUID listId, UUID itemId, Family family) {
        SharedList list = getListOrThrow(listId, family);
        list.getItems().removeIf(item -> item.getId().equals(itemId));
        sharedListRepository.save(list);
    }

    @Transactional
    public ClearCompletedResponse clearCompleted(UUID listId, Family family) {
        SharedList list = getListOrThrow(listId, family);
        int before = list.getItems().size();
        list.getItems().removeIf(SharedListItem::isCompleted);
        sharedListRepository.save(list);
        return new ClearCompletedResponse(before - list.getItems().size());
    }

    public ListPreferencesResponse getPreferences(Family family) {
        ListPreferences preferences = listPreferencesRepository.findByFamily(family)
                .orElseThrow(() -> new ResourceNotFoundException("List Preferences", family.getId()));
        return new ListPreferencesResponse(preferences.isShowCompletedByDefault());
    }

    @Transactional
    public ListPreferencesResponse updatePreferences(UpdateListPreferencesRequest request, Family family) {
        ListPreferences preferences = listPreferencesRepository.findByFamily(family)
                .orElseThrow(() -> new ResourceNotFoundException("List Preferences", family.getId()));
        preferences.setShowCompletedByDefault(request.showCompletedByDefault());
        return new ListPreferencesResponse(listPreferencesRepository.save(preferences).isShowCompletedByDefault());
    }

    private ListCategory resolveCategory(SharedList list, UUID categoryId) {
        if (list.getKind() == ListKind.GENERAL) {
            if (categoryId != null) {
                throw new BadRequestException("General lists do not support categories.");
            }
            return null;
        }
        if (categoryId == null) {
            return null;
        }
        ListCategory category = listCategoryRepository.findByFamilyAndId(list.getFamily(), categoryId)
                .orElseThrow(() -> new ResourceNotFoundException("List Category", categoryId));
        if (category.getKind() != list.getKind()) {
            throw new BadRequestException("Category kind does not match list kind.");
        }
        return category;
    }

    private SharedList getListOrThrow(UUID id, Family family) {
        return sharedListRepository.findDetailByFamilyAndId(family, id)
                .orElseThrow(() -> new ResourceNotFoundException("List", id));
    }

    private ListDetailResponse mapDetail(SharedList list) {
        List<ListCategoryResponse> categories = list.getKind() == ListKind.GENERAL
                ? List.of()
                : listCategoryRepository.findByFamilyAndKindOrderBySortOrderAsc(list.getFamily(), list.getKind())
                        .stream()
                        .map(ListMapper::toCategoryDto)
                        .toList();

        return ListMapper.toDetailDto(list, categories);
    }
}
```

```java
@RestController
@RequestMapping("/api/lists")
@RequiredArgsConstructor
public class ListController {
    private final ListService listService;

    @GetMapping
    public ResponseEntity<ApiResponse<List<ListSummaryResponse>>> getLists(@AuthenticationPrincipal Family family) {
        return ResponseEntity.ok(new ApiResponse<>(listService.getLists(family), ""));
    }

    @PostMapping
    public ResponseEntity<ApiResponse<ListDetailResponse>> createList(
            @Valid @RequestBody CreateListRequest request,
            @AuthenticationPrincipal Family family
    ) {
        ListDetailResponse response = listService.createList(request, family);
        return ResponseEntity.created(URI.create("/api/lists/" + response.id()))
                .body(new ApiResponse<>(response, "List created successfully"));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<ListDetailResponse>> getList(
            @PathVariable UUID id,
            @AuthenticationPrincipal Family family
    ) {
        return ResponseEntity.ok(new ApiResponse<>(listService.getList(id, family), ""));
    }

    @PatchMapping("/{id}")
    public ResponseEntity<ApiResponse<ListDetailResponse>> updateList(
            @PathVariable UUID id,
            @Valid @RequestBody UpdateListRequest request,
            @AuthenticationPrincipal Family family
    ) {
        return ResponseEntity.ok(new ApiResponse<>(listService.updateList(id, request, family), "List updated successfully"));
    }

    @PostMapping("/{id}/items")
    public ResponseEntity<ApiResponse<ListItemResponse>> createItem(
            @PathVariable UUID id,
            @Valid @RequestBody CreateListItemRequest request,
            @AuthenticationPrincipal Family family
    ) {
        ListItemResponse response = listService.createItem(id, request, family);
        return ResponseEntity.created(URI.create("/api/lists/" + id + "/items/" + response.id()))
                .body(new ApiResponse<>(response, "List item created successfully"));
    }

    @PatchMapping("/{listId}/items/{itemId}")
    public ResponseEntity<ApiResponse<ListItemResponse>> updateItem(
            @PathVariable UUID listId,
            @PathVariable UUID itemId,
            @Valid @RequestBody UpdateListItemRequest request,
            @AuthenticationPrincipal Family family
    ) {
        return ResponseEntity.ok(
                new ApiResponse<>(listService.updateItem(listId, itemId, request, family), "List item updated successfully")
        );
    }

    @DeleteMapping("/{listId}/items/{itemId}")
    public ResponseEntity<Void> deleteItem(
            @PathVariable UUID listId,
            @PathVariable UUID itemId,
            @AuthenticationPrincipal Family family
    ) {
        listService.deleteItem(listId, itemId, family);
        return ResponseEntity.noContent().build();
    }

    @PostMapping("/{id}/clear-completed")
    public ResponseEntity<ApiResponse<ClearCompletedResponse>> clearCompleted(
            @PathVariable UUID id,
            @AuthenticationPrincipal Family family
    ) {
        return ResponseEntity.ok(
                new ApiResponse<>(listService.clearCompleted(id, family), "Completed items removed successfully")
        );
    }

    @GetMapping("/preferences")
    public ResponseEntity<ApiResponse<ListPreferencesResponse>> getPreferences(@AuthenticationPrincipal Family family) {
        return ResponseEntity.ok(new ApiResponse<>(listService.getPreferences(family), ""));
    }

    @PatchMapping("/preferences")
    public ResponseEntity<ApiResponse<ListPreferencesResponse>> updatePreferences(
            @Valid @RequestBody UpdateListPreferencesRequest request,
            @AuthenticationPrincipal Family family
    ) {
        return ResponseEntity.ok(
                new ApiResponse<>(listService.updatePreferences(request, family), "List preferences updated successfully")
        );
    }
}
```

- [ ] **Step 4: Run the backend controller and service tests**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=ListServiceTest,ListControllerTest test
```

Expected: PASS for category validation, clear-completed behavior, and controller contract tests.

- [ ] **Step 5: Commit**

```bash
cd backend/family-hub-api
git add src/main/java/com/familyhub/demo/dto/CreateListRequest.java \
  src/main/java/com/familyhub/demo/dto/UpdateListRequest.java \
  src/main/java/com/familyhub/demo/dto/CreateListItemRequest.java \
  src/main/java/com/familyhub/demo/dto/UpdateListItemRequest.java \
  src/main/java/com/familyhub/demo/dto/ListSummaryResponse.java \
  src/main/java/com/familyhub/demo/dto/ListDetailResponse.java \
  src/main/java/com/familyhub/demo/dto/ListItemResponse.java \
  src/main/java/com/familyhub/demo/dto/ListCategoryResponse.java \
  src/main/java/com/familyhub/demo/dto/ListPreferencesResponse.java \
  src/main/java/com/familyhub/demo/dto/UpdateListPreferencesRequest.java \
  src/main/java/com/familyhub/demo/dto/ClearCompletedResponse.java \
  src/main/java/com/familyhub/demo/mapper/ListMapper.java \
  src/main/java/com/familyhub/demo/repository/SharedListRepository.java \
  src/main/java/com/familyhub/demo/service/ListService.java \
  src/main/java/com/familyhub/demo/controller/ListController.java \
  src/test/java/com/familyhub/demo/TestDataFactory.java \
  src/test/java/com/familyhub/demo/service/ListServiceTest.java \
  src/test/java/com/familyhub/demo/controller/ListControllerTest.java
git commit -m "feat(lists): add backend list endpoints"
```

## Task 3: Verify Backend End-To-End And Publish The Release Handoff

**Files:**
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/integration/ListIntegrationTest.java`

- [ ] **Step 1: Write the failing integration test that exercises auth, list create, list detail, item updates, and clear-completed**

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
class ListIntegrationTest {

    @Autowired
    MockMvc mockMvc;

    @Test
    void groceryListFlow_persistsCategoriesAndClearCompleted() throws Exception {
        String authBody = mockMvc.perform(post("/api/auth/register")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {
                                    "username": "listsfamily",
                                    "password": "TestPassword123!",
                                    "familyName": "Lists Family",
                                    "members": [
                                        { "name": "Alice", "color": "coral" }
                                    ]
                                }
                                """))
                .andExpect(status().isOk())
                .andReturn()
                .getResponse()
                .getContentAsString();

        String token = JsonPath.read(authBody, "$.data.token");

        String listBody = mockMvc.perform(post("/api/lists")
                        .header("Authorization", "Bearer " + token)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {
                                    "name": "Trader Joe's Run",
                                    "kind": "grocery"
                                }
                                """))
                .andExpect(status().isCreated())
                .andReturn()
                .getResponse()
                .getContentAsString();

        String listId = JsonPath.read(listBody, "$.data.id");

        String detailBody = mockMvc.perform(get("/api/lists/{id}", listId)
                        .header("Authorization", "Bearer " + token))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.categories[0].name").value("Produce"))
                .andReturn()
                .getResponse()
                .getContentAsString();

        String produceCategoryId = JsonPath.read(detailBody, "$.data.categories[0].id");

        String itemBody = mockMvc.perform(post("/api/lists/{id}/items", listId)
                        .header("Authorization", "Bearer " + token)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {
                                    "text": "Bananas",
                                    "categoryId": "%s"
                                }
                                """.formatted(produceCategoryId)))
                .andExpect(status().isCreated())
                .andReturn()
                .getResponse()
                .getContentAsString();

        String itemId = JsonPath.read(itemBody, "$.data.id");

        mockMvc.perform(patch("/api/lists/{listId}/items/{itemId}", listId, itemId)
                        .header("Authorization", "Bearer " + token)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {
                                    "text": "Bananas",
                                    "completed": true,
                                    "categoryId": "%s"
                                }
                                """.formatted(produceCategoryId)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.completed").value(true));

        mockMvc.perform(post("/api/lists/{id}/clear-completed", listId)
                        .header("Authorization", "Bearer " + token))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.removedCount").value(1));
    }
}
```

- [ ] **Step 2: Run the backend integration test to verify the full flow passes**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=ListIntegrationTest test
```

Expected: PASS with the full authenticated grocery-list flow against the real test database.

- [ ] **Step 3: Run the full backend test suite**

Run:

```bash
cd backend/family-hub-api
./mvnw test
```

Expected: PASS for all backend tests, including the new lists unit and integration coverage.

- [ ] **Step 4: After the backend PR merges, verify the published release exists before FE live verification**

Run:

```bash
gh api repos/joe-bor/family-hub-api/releases/latest \
  --jq '{tag:.tag_name,name:.name,published:.published_at}'
```

Expected: JSON object showing the latest published backend release that contains `/api/lists`.

- [ ] **Step 5: Commit**

```bash
cd backend/family-hub-api
git add src/test/java/com/familyhub/demo/integration/ListIntegrationTest.java
git commit -m "test(lists): add backend list integration coverage"
```

## Task 4: Add Frontend Lists Types, Validation, Services, Hooks, And MSW Mocks

**Files:**
- Create: `frontend/src/lib/types/lists.ts`
- Modify: `frontend/src/lib/types/index.ts`
- Create: `frontend/src/api/services/lists.service.ts`
- Modify: `frontend/src/api/services/index.ts`
- Create: `frontend/src/api/hooks/use-lists.ts`
- Modify: `frontend/src/api/hooks/index.ts`
- Modify: `frontend/src/api/index.ts`
- Create: `frontend/src/lib/validations/lists.ts`
- Create: `frontend/src/lib/validations/lists.test.ts`
- Modify: `frontend/src/lib/validations/index.ts`
- Modify: `frontend/src/test/mocks/handlers.ts`
- Modify: `frontend/src/test/mocks/server.ts`
- Delete: `frontend/src/stores/lists-store.ts`
- Modify: `frontend/src/stores/index.ts`
- Create: `frontend/src/api/hooks/use-lists.test.tsx`

- [ ] **Step 1: Write the failing validation and hook tests**

```ts
describe("listCreateSchema", () => {
  it("trims name and requires a kind", () => {
    expect(
      listCreateSchema.parse({ name: "  Trader Joe's Run  ", kind: "grocery" }),
    ).toEqual({
      name: "Trader Joe's Run",
      kind: "grocery",
    });
  });

  it("rejects empty names", () => {
    expect(() =>
      listCreateSchema.parse({ name: "   ", kind: "grocery" }),
    ).toThrow("List name is required");
  });
});

describe("useLists", () => {
  it("returns hub summaries from the API", async () => {
    seedMockLists([
      {
        id: "list-1",
        name: "Trader Joe's Run",
        kind: "grocery",
        categoryDisplayMode: "grouped",
        showCompletedOverride: null,
        categories: [],
        items: [],
        createdAt: "2026-05-06T09:00:00.000Z",
        updatedAt: "2026-05-06T09:00:00.000Z",
      },
    ]);
    const { result } = renderHook(() => useLists(), { wrapper: createWrapper() });

    await waitFor(() => {
      expect(result.current.data?.data[0].name).toBe("Trader Joe's Run");
    });
  });

  it("optimistically updates completed visibility preference", async () => {
    seedMockLists([]);
    seedMockListPreferences({ showCompletedByDefault: true });

    const { result } = renderHook(() => useUpdateListPreferences(), {
      wrapper: createWrapper(),
    });

    result.current.mutate({ showCompletedByDefault: false });

    await waitFor(() => {
      expect(
        queryClient.getQueryData<ListPreferencesApiResponse>(listsKeys.preferences())?.data.showCompletedByDefault,
      ).toBe(false);
    });
  });
});
```

- [ ] **Step 2: Run the targeted frontend tests to verify they fail before implementation**

Run:

```bash
cd frontend
npm run test -- --run src/lib/validations/lists.test.ts src/api/hooks/use-lists.test.tsx
```

Expected: FAIL because the lists types, hooks, and validation modules do not exist yet.

- [ ] **Step 3: Add the wire types, schemas, services, hooks, and MSW handlers**

```ts
export type ListKind = "grocery" | "to-do" | "general";
export type ListCategoryDisplayMode = "grouped" | "flat";

export interface ListSummary {
  id: string;
  name: string;
  kind: ListKind;
  totalItems: number;
  completedItems: number;
}

export interface ListCategory {
  id: string;
  kind: Exclude<ListKind, "general">;
  name: string;
  seeded: boolean;
  sortOrder: number;
}

export interface ListItem {
  id: string;
  text: string;
  completed: boolean;
  completedAt: string | null;
  categoryId: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface ListDetail {
  id: string;
  name: string;
  kind: ListKind;
  categoryDisplayMode: ListCategoryDisplayMode;
  showCompletedOverride: boolean | null;
  categories: ListCategory[];
  items: ListItem[];
  createdAt: string;
  updatedAt: string;
}

export interface ListPreferences {
  showCompletedByDefault: boolean;
}

export interface CreateListRequest {
  name: string;
  kind: ListKind;
}

export interface UpdateListRequest {
  categoryDisplayMode: ListCategoryDisplayMode;
  showCompletedOverride: boolean | null;
}

export interface CreateListItemRequest {
  text: string;
  categoryId: string | null;
}

export interface UpdateListItemRequest {
  text: string;
  completed: boolean;
  categoryId: string | null;
}

export interface UpdateListPreferencesRequest {
  showCompletedByDefault: boolean;
}

export type ListSummariesApiResponse = ApiResponse<ListSummary[]>;
export type ListDetailApiResponse = ApiResponse<ListDetail>;
export type ListItemApiResponse = ApiResponse<ListItem>;
export type ListPreferencesApiResponse = ApiResponse<ListPreferences>;
export type ClearCompletedApiResponse = ApiResponse<{ removedCount: number }>;
```

```ts
export const listCreateSchema = z.object({
  name: z
    .string()
    .transform((value) => value.trim())
    .pipe(
      z
        .string()
        .min(1, "List name is required")
        .max(100, "List name must be 100 characters or less"),
    ),
  kind: z.enum(["grocery", "to-do", "general"]),
});

export const listItemSchema = z.object({
  text: z
    .string()
    .transform((value) => value.trim())
    .pipe(
      z
        .string()
        .min(1, "Item text is required")
        .max(100, "Item text must be 100 characters or less"),
    ),
  categoryId: z.string().uuid().nullable().optional(),
});

export type ListCreateFormData = z.infer<typeof listCreateSchema>;
export type ListItemFormData = z.infer<typeof listItemSchema>;
```

```ts
export const listsService = {
  getLists: () => httpClient.get<ListSummariesApiResponse>("/lists"),
  getList: (id: string) => httpClient.get<ListDetailApiResponse>(`/lists/${id}`),
  createList: (request: CreateListRequest) =>
    httpClient.post<ListDetailApiResponse>("/lists", request),
  updateList: (id: string, request: UpdateListRequest) =>
    httpClient.patch<ListDetailApiResponse>(`/lists/${id}`, request),
  createItem: (listId: string, request: CreateListItemRequest) =>
    httpClient.post<ListItemApiResponse>(`/lists/${listId}/items`, request),
  updateItem: (listId: string, itemId: string, request: UpdateListItemRequest) =>
    httpClient.patch<ListItemApiResponse>(`/lists/${listId}/items/${itemId}`, request),
  deleteItem: (listId: string, itemId: string) =>
    httpClient.delete(`/lists/${listId}/items/${itemId}`),
  clearCompleted: (listId: string) =>
    httpClient.post<ClearCompletedApiResponse>(`/lists/${listId}/clear-completed`, {}),
  getPreferences: () => httpClient.get<ListPreferencesApiResponse>("/lists/preferences"),
  updatePreferences: (request: UpdateListPreferencesRequest) =>
    httpClient.patch<ListPreferencesApiResponse>("/lists/preferences", request),
};
```

```ts
export const listsKeys = {
  all: ["lists"] as const,
  hub: () => [...listsKeys.all, "hub"] as const,
  detail: (id: string) => [...listsKeys.all, "detail", id] as const,
  preferences: () => [...listsKeys.all, "preferences"] as const,
};

export function useLists() {
  return useQuery({
    queryKey: listsKeys.hub(),
    queryFn: listsService.getLists,
  });
}

export function useList(id: string | null) {
  return useQuery({
    queryKey: listsKeys.detail(id ?? "none"),
    queryFn: () => listsService.getList(id!),
    enabled: id !== null,
  });
}

export function useListPreferences() {
  return useQuery({
    queryKey: listsKeys.preferences(),
    queryFn: listsService.getPreferences,
  });
}

export function useCreateList(options?: { onSuccess?: (data: ListDetailApiResponse) => void }) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: listsService.createList,
    onSuccess: (response) => {
      queryClient.invalidateQueries({ queryKey: listsKeys.hub() });
      queryClient.setQueryData(listsKeys.detail(response.data.id), response);
      options?.onSuccess?.(response);
    },
  });
}

export function useUpdateList(id: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (request: UpdateListRequest) => listsService.updateList(id, request),
    onSuccess: (response) => {
      queryClient.invalidateQueries({ queryKey: listsKeys.hub() });
      queryClient.setQueryData(listsKeys.detail(id), response);
    },
  });
}

export function useCreateListItem(listId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (request: CreateListItemRequest) => listsService.createItem(listId, request),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: listsKeys.hub() });
      queryClient.invalidateQueries({ queryKey: listsKeys.detail(listId) });
    },
  });
}

export function useUpdateListItem(listId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({
      itemId,
      request,
    }: {
      itemId: string;
      request: UpdateListItemRequest;
    }) => listsService.updateItem(listId, itemId, request),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: listsKeys.hub() });
      queryClient.invalidateQueries({ queryKey: listsKeys.detail(listId) });
    },
  });
}

export function useDeleteListItem(listId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (itemId: string) => listsService.deleteItem(listId, itemId),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: listsKeys.hub() });
      queryClient.invalidateQueries({ queryKey: listsKeys.detail(listId) });
    },
  });
}

export function useClearCompleted(listId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: () => listsService.clearCompleted(listId),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: listsKeys.hub() });
      queryClient.invalidateQueries({ queryKey: listsKeys.detail(listId) });
    },
  });
}

export function useUpdateListPreferences() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: listsService.updatePreferences,
    onMutate: async (request) => {
      await queryClient.cancelQueries({ queryKey: listsKeys.preferences() });
      const previous = queryClient.getQueryData<ListPreferencesApiResponse>(listsKeys.preferences());
      queryClient.setQueryData<ListPreferencesApiResponse>(listsKeys.preferences(), {
        data: request,
      });
      return { previous };
    },
    onError: (_error, _request, context) => {
      if (context?.previous) {
        queryClient.setQueryData(listsKeys.preferences(), context.previous);
      }
    },
  });
}
```

```ts
let mockLists: ListDetail[] = [];
let mockListPreferences: ListPreferences = { showCompletedByDefault: true };

export function resetMockLists(): void {
  mockLists = [];
  mockListPreferences = { showCompletedByDefault: true };
}

export function seedMockLists(lists: ListDetail[]): void {
  mockLists = [...lists];
}

export function seedMockListPreferences(preferences: ListPreferences): void {
  mockListPreferences = preferences;
}

http.get(`${API_BASE}/lists`, () => {
  return HttpResponse.json(
    createApiResponse(
      mockLists.map((list) => ({
        id: list.id,
        name: list.name,
        kind: list.kind,
        totalItems: list.items.length,
        completedItems: list.items.filter((item) => item.completed).length,
      })),
    ),
  );
});

export {
  resetMockLists,
  seedMockListPreferences,
  seedMockLists,
};
```

- [ ] **Step 4: Re-run the targeted frontend tests and remove the local lists store export**

Run:

```bash
cd frontend
npm run test -- --run src/lib/validations/lists.test.ts src/api/hooks/use-lists.test.tsx
```

Expected: PASS, and `src/stores/index.ts` no longer exports `useListsStore`.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/lib/types/lists.ts \
  src/lib/types/index.ts \
  src/api/services/lists.service.ts \
  src/api/services/index.ts \
  src/api/hooks/use-lists.ts \
  src/api/hooks/index.ts \
  src/api/index.ts \
  src/lib/validations/lists.ts \
  src/lib/validations/lists.test.ts \
  src/lib/validations/index.ts \
  src/test/mocks/handlers.ts \
  src/test/mocks/server.ts \
  src/api/hooks/use-lists.test.tsx \
  src/stores/index.ts
git rm src/stores/lists-store.ts
git commit -m "feat(lists): add frontend lists data hooks"
```

## Task 5: Build The Lists Hub And List-Creation Flow

**Files:**
- Create: `frontend/src/components/ui/mobile-sheet.tsx`
- Modify: `frontend/src/components/calendar/components/mobile-event-sheet.tsx`
- Create: `frontend/src/components/lists/list-create-sheet.tsx`
- Create: `frontend/src/components/lists/list-card.tsx`
- Modify: `frontend/src/components/lists-view.tsx`
- Create: `frontend/src/components/lists-view.test.tsx`

- [ ] **Step 1: Write the failing component tests for the hub and create-list flow**

```tsx
describe("ListsView hub", () => {
  it("renders persisted list summaries instead of placeholder cards", async () => {
    seedFamilyStore({ name: "Test Family", members: [{ id: "1", name: "Alice", color: "coral" }] });
    seedMockLists([
      {
        id: "list-1",
        name: "Trader Joe's Run",
        kind: "grocery",
        categoryDisplayMode: "grouped",
        showCompletedOverride: null,
        categories: [],
        items: [],
        createdAt: "2026-05-06T09:00:00.000Z",
        updatedAt: "2026-05-06T09:00:00.000Z",
      },
      {
        id: "list-2",
        name: "Weekend Reset",
        kind: "to-do",
        categoryDisplayMode: "grouped",
        showCompletedOverride: null,
        categories: [],
        items: [],
        createdAt: "2026-05-06T09:05:00.000Z",
        updatedAt: "2026-05-06T09:05:00.000Z",
      },
    ]);

    render(<ListsView />);

    expect(await screen.findByRole("heading", { name: "My Lists" })).toBeInTheDocument();
    expect(screen.getByRole("button", { name: /Trader Joe's Run/i })).toBeInTheDocument();
    expect(screen.getByRole("button", { name: /Weekend Reset/i })).toBeInTheDocument();
  });

  it("creates a new grocery list through the mobile sheet flow", async () => {
    seedFamilyStore({ name: "Test Family", members: [{ id: "1", name: "Alice", color: "coral" }] });
    seedMockLists([]);

    const { user } = renderWithUser(<ListsView />);

    await user.click(screen.getByRole("button", { name: /new list/i }));
    await user.type(screen.getByLabelText("List name"), "Target Run");
    await user.click(screen.getByRole("radio", { name: "Grocery" }));
    await user.click(screen.getByRole("button", { name: "Create list" }));

    expect(await screen.findByRole("heading", { name: "Target Run" })).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run the hub test to verify it fails against the placeholder `lists-view`**

Run:

```bash
cd frontend
npm run test -- --run src/components/lists-view.test.tsx
```

Expected: FAIL because `ListsView` still renders hard-coded sample cards and has no create flow.

- [ ] **Step 3: Extract the shared mobile sheet and replace the placeholder hub**

```tsx
interface MobileSheetProps {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  headerRight?: ReactNode;
  children: ReactNode;
}

export function MobileSheet({
  isOpen,
  onClose,
  title,
  headerRight,
  children,
}: MobileSheetProps) {
  if (!isOpen) return null;

  return (
    <div
      role="dialog"
      aria-label={title}
      aria-describedby={undefined}
      className={cn(
        "fixed inset-0 z-50 flex flex-col bg-card",
        "motion-safe:animate-in motion-safe:slide-in-from-bottom motion-safe:duration-200",
      )}
    >
      <div className="flex items-center justify-between border-b border-border px-4 py-3">
        <button type="button" onClick={onClose} className="rounded-lg px-1 py-1 text-sm font-semibold text-primary">
          Cancel
        </button>
        <h2 className="text-[20px] leading-7 font-semibold">{title}</h2>
        {headerRight ?? <div className="w-16" />}
      </div>
      <div className="flex-1 overflow-y-auto px-4 py-5">{children}</div>
    </div>
  );
}
```

```tsx
export function ListCreateSheet({
  open,
  onOpenChange,
  onCreated,
}: {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  onCreated: (id: string) => void;
}) {
  const createList = useCreateList({
    onSuccess: (response) => {
      onOpenChange(false);
      onCreated(response.data.id);
    },
  });
  const form = useForm<ListCreateFormData>({
    resolver: zodResolver(listCreateSchema),
    defaultValues: { name: "", kind: "general" },
  });

  return (
    <MobileSheet
      isOpen={open}
      onClose={() => onOpenChange(false)}
      title="New List"
      headerRight={
        <button
          type="button"
          onClick={form.handleSubmit((values) => createList.mutate(values))}
          className="rounded-lg px-1 py-1 text-sm font-semibold text-primary"
        >
          Create list
        </button>
      }
    >
      <form className="space-y-6">
        <div className="space-y-2">
          <Label htmlFor="list-name">List name</Label>
          <Input id="list-name" {...form.register("name")} />
        </div>
        <fieldset className="space-y-3">
          <legend className="text-sm font-semibold text-foreground">List kind</legend>
          {[
            ["grocery", "Grocery", "Built-in shopping categories"],
            ["to-do", "To-do", "Urgency buckets like Urgent, Soon, Later"],
            ["general", "General", "Flat checklist with no categories"],
          ].map(([value, label, description]) => (
            <label
              key={value}
              className="flex items-start gap-3 rounded-xl border border-border p-3"
            >
              <input
                type="radio"
                value={value}
                checked={form.watch("kind") === value}
                onChange={() => form.setValue("kind", value as ListKind)}
                aria-label={label}
              />
              <span>
                <span className="block font-medium text-foreground">{label}</span>
                <span className="block text-sm text-muted-foreground">{description}</span>
              </span>
            </label>
          ))}
        </fieldset>
      </form>
    </MobileSheet>
  );
}
```

```tsx
export function ListsView() {
  const [selectedListId, setSelectedListId] = useState<string | null>(null);
  const [createOpen, setCreateOpen] = useState(false);
  const lists = useLists();
  const preferences = useListPreferences();

  if (selectedListId !== null) {
    return (
      <ListDetailView
        listId={selectedListId}
        preferences={preferences.data?.data ?? { showCompletedByDefault: true }}
        onBack={() => setSelectedListId(null)}
      />
    );
  }

  const summaries = lists.data?.data ?? [];

  return (
    <div className="flex-1 overflow-y-auto p-4 sm:p-6">
      <div className="mx-auto max-w-2xl space-y-6">
        <div className="flex items-center justify-between gap-3">
          <h2 className="text-[24px] leading-8 font-semibold text-foreground">My Lists</h2>
          <Button onClick={() => setCreateOpen(true)}>New List</Button>
        </div>

        {summaries.length === 0 ? (
          <div className="rounded-2xl bg-card p-6 text-center shadow-sm">
            <h3 className="text-lg font-semibold text-foreground">No lists yet</h3>
            <p className="mt-2 text-sm text-muted-foreground">
              Create your first family list for groceries, to-dos, or anything else.
            </p>
          </div>
        ) : (
          <div className="grid grid-cols-2 gap-3 sm:gap-4">
            {summaries.map((list) => (
              <ListCard key={list.id} list={list} onOpen={() => setSelectedListId(list.id)} />
            ))}
          </div>
        )}

        <ListCreateSheet
          open={createOpen}
          onOpenChange={setCreateOpen}
          onCreated={(id) => setSelectedListId(id)}
        />
      </div>
    </div>
  );
}
```

- [ ] **Step 4: Run the component test and one calendar test to verify the shared sheet extraction did not break Calendar**

Run:

```bash
cd frontend
npm run test -- --run src/components/lists-view.test.tsx src/api/hooks/use-calendar-put.test.tsx
```

Expected: PASS for the new hub/create flow and no regression in the existing calendar modal path.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/components/ui/mobile-sheet.tsx \
  src/components/calendar/components/mobile-event-sheet.tsx \
  src/components/lists/list-create-sheet.tsx \
  src/components/lists/list-card.tsx \
  src/components/lists-view.tsx \
  src/components/lists-view.test.tsx
git commit -m "feat(lists): add lists hub and create flow"
```

## Task 6: Build List Detail Interactions, Grouping Logic, And Completed Controls

**Files:**
- Create: `frontend/src/components/lists/list-item-sheet.tsx`
- Create: `frontend/src/components/lists/list-detail-view.tsx`
- Create: `frontend/src/components/lists/list-item-row.tsx`
- Create: `frontend/src/components/lists/build-list-sections.ts`
- Create: `frontend/src/components/lists/build-list-sections.test.ts`
- Modify: `frontend/src/components/lists-view.tsx`

- [ ] **Step 1: Write the failing grouping and completed-visibility tests**

```ts
describe("buildListSections", () => {
  it("groups grocery items by seeded categories and keeps completed items last within a section", () => {
    const sections = buildListSections({
      list: {
        id: "list-1",
        name: "Trader Joe's Run",
        kind: "grocery",
        categoryDisplayMode: "grouped",
        showCompletedOverride: null,
        categories: [
          { id: "produce", kind: "grocery", name: "Produce", seeded: true, sortOrder: 0 },
        ],
        items: [
          { id: "1", text: "Bananas", completed: false, completedAt: null, categoryId: "produce", createdAt: "2026-05-06T09:00:00.000Z", updatedAt: "2026-05-06T09:00:00.000Z" },
          { id: "2", text: "Spinach", completed: true, completedAt: "2026-05-06T10:00:00.000Z", categoryId: "produce", createdAt: "2026-05-06T09:05:00.000Z", updatedAt: "2026-05-06T10:00:00.000Z" },
          { id: "3", text: "Paper towels", completed: false, completedAt: null, categoryId: null, createdAt: "2026-05-06T09:10:00.000Z", updatedAt: "2026-05-06T09:10:00.000Z" },
        ],
        createdAt: "2026-05-06T09:00:00.000Z",
        updatedAt: "2026-05-06T09:10:00.000Z",
      },
      showCompleted: true,
    });

    expect(sections.map((section) => section.title)).toEqual(["Produce", "Uncategorized"]);
    expect(sections[0].items.map((item) => item.text)).toEqual(["Bananas", "Spinach"]);
  });

  it("hides completed items when visibility resolves to false", () => {
    const sections = buildListSections({
      list: {
        id: "list-2",
        name: "Weekend Reset",
        kind: "to-do",
        categoryDisplayMode: "flat",
        showCompletedOverride: null,
        categories: [],
        items: [
          { id: "1", text: "Pay bills", completed: true, completedAt: "2026-05-06T10:00:00.000Z", categoryId: null, createdAt: "2026-05-06T09:00:00.000Z", updatedAt: "2026-05-06T10:00:00.000Z" },
          { id: "2", text: "Call dentist", completed: false, completedAt: null, categoryId: null, createdAt: "2026-05-06T09:05:00.000Z", updatedAt: "2026-05-06T09:05:00.000Z" },
        ],
        createdAt: "2026-05-06T09:00:00.000Z",
        updatedAt: "2026-05-06T10:00:00.000Z",
      },
      showCompleted: false,
    });

    expect(sections[0].items.map((item) => item.text)).toEqual(["Call dentist"]);
  });
});
```

```tsx
it("supports per-list completed override and clear completed", async () => {
  seedFamilyStore({ name: "Test Family", members: [{ id: "1", name: "Alice", color: "coral" }] });
  seedMockLists([
    {
      id: "list-2",
      name: "Weekend Reset",
      kind: "to-do",
      categoryDisplayMode: "flat",
      showCompletedOverride: null,
      categories: [],
      items: [
        { id: "item-1", text: "Pay bills", completed: true, completedAt: "2026-05-06T10:00:00.000Z", categoryId: null, createdAt: "2026-05-06T09:00:00.000Z", updatedAt: "2026-05-06T10:00:00.000Z" },
        { id: "item-2", text: "Call dentist", completed: false, completedAt: null, categoryId: null, createdAt: "2026-05-06T09:05:00.000Z", updatedAt: "2026-05-06T09:05:00.000Z" },
      ],
      createdAt: "2026-05-06T09:00:00.000Z",
      updatedAt: "2026-05-06T10:00:00.000Z",
    },
  ]);
  seedMockListPreferences({ showCompletedByDefault: true });

  const { user } = renderWithUser(<ListsView />);

  await user.click(await screen.findByRole("button", { name: /Weekend Reset/i }));
  await user.selectOptions(screen.getByLabelText("Completed items"), "hide");

  expect(screen.queryByText("Pay bills")).not.toBeInTheDocument();

  await user.click(screen.getByRole("button", { name: /remove all completed/i }));

  await waitFor(() => {
    expect(screen.queryByText("Pay bills")).not.toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run the grouping and detail tests to verify they fail before implementation**

Run:

```bash
cd frontend
npm run test -- --run src/components/lists/build-list-sections.test.ts src/components/lists-view.test.tsx
```

Expected: FAIL because no grouped/flat helper or completed-override UI exists yet.

- [ ] **Step 3: Add the section builder, item sheet, detail view, and completed controls**

```ts
export interface BuiltListSection {
  id: string;
  title: string | null;
  items: ListItem[];
}

function sortVisibleItems(items: ListItem[]): ListItem[] {
  return [...items].sort((left, right) => {
    if (left.completed !== right.completed) {
      return Number(left.completed) - Number(right.completed);
    }
    return left.createdAt.localeCompare(right.createdAt);
  });
}

export function buildListSections({
  list,
  showCompleted,
}: {
  list: ListDetail;
  showCompleted: boolean;
}): BuiltListSection[] {
  const visibleItems = showCompleted
    ? list.items
    : list.items.filter((item) => !item.completed);

  if (list.kind === "general" || list.categoryDisplayMode === "flat") {
    return [{ id: "all", title: null, items: sortVisibleItems(visibleItems) }];
  }

  const sections = list.categories.map((category) => ({
    id: category.id,
    title: category.name,
    items: sortVisibleItems(visibleItems.filter((item) => item.categoryId === category.id)),
  }));

  const uncategorized = sortVisibleItems(
    visibleItems.filter((item) => item.categoryId === null),
  );

  return [
    ...sections.filter((section) => section.items.length > 0),
    ...(uncategorized.length > 0
      ? [{ id: "uncategorized", title: "Uncategorized", items: uncategorized }]
      : []),
  ];
}
```

```tsx
export function ListDetailView({
  listId,
  preferences,
  onBack,
}: {
  listId: string;
  preferences: ListPreferences;
  onBack: () => void;
}) {
  const { data } = useList(listId);
  const updateList = useUpdateList(listId);
  const updateItem = useUpdateListItem(listId);
  const deleteItem = useDeleteListItem(listId);
  const updatePreferences = useUpdateListPreferences();
  const clearCompleted = useClearCompleted(listId);
  const [itemSheet, setItemSheet] = useState<{ mode: "create" | "edit"; itemId?: string } | null>(null);

  const list = data?.data;
  if (!list) {
    return <div className="flex-1 p-4 text-sm text-muted-foreground">Loading list…</div>;
  }

  const kindLabel =
    list.kind === "to-do"
      ? "To-do"
      : list.kind === "grocery"
        ? "Grocery"
        : "General";

  const resolvedShowCompleted =
    list.showCompletedOverride ?? preferences.showCompletedByDefault;

  const sections = buildListSections({
    list,
    showCompleted: resolvedShowCompleted,
  });

  return (
    <div className="flex-1 overflow-y-auto p-4 sm:p-6">
      <div className="mx-auto max-w-2xl space-y-4">
        <button onClick={onBack} className="rounded-lg py-1 text-sm font-semibold text-primary hover:underline">
          ← Back to Lists
        </button>

        <div className="rounded-2xl bg-card p-4 shadow-sm">
          <div className="flex items-center justify-between gap-3">
            <div>
              <p className="text-xs font-semibold uppercase tracking-wide text-muted-foreground">
                {kindLabel}
              </p>
              <h2 className="text-[22px] leading-7 font-semibold text-foreground">{list.name}</h2>
            </div>
            <Button onClick={() => setItemSheet({ mode: "create" })}>Add item</Button>
          </div>

          <div className="mt-4 grid gap-3 sm:grid-cols-2">
            {list.kind !== "general" && (
              <div className="space-y-1">
                <Label htmlFor="category-mode">Categories</Label>
                <select
                  id="category-mode"
                  value={list.categoryDisplayMode}
                  onChange={(event) =>
                    updateList.mutate({
                      categoryDisplayMode: event.target.value as ListCategoryDisplayMode,
                      showCompletedOverride: list.showCompletedOverride,
                    })
                  }
                  className="h-10 w-full rounded-lg border border-border bg-background px-3"
                >
                  <option value="grouped">Show categories</option>
                  <option value="flat">Hide categories</option>
                </select>
              </div>
            )}

            <div className="space-y-1">
              <Label htmlFor="completed-items">Completed items</Label>
              <select
                id="completed-items"
                aria-label="Completed items"
                value={
                  list.showCompletedOverride === null
                    ? "family-default"
                    : list.showCompletedOverride
                      ? "show"
                      : "hide"
                }
                onChange={(event) =>
                  updateList.mutate({
                    categoryDisplayMode: list.categoryDisplayMode,
                    showCompletedOverride:
                      event.target.value === "family-default"
                        ? null
                        : event.target.value === "show",
                  })
                }
                className="h-10 w-full rounded-lg border border-border bg-background px-3"
              >
                <option value="family-default">
                    Family default ({preferences.showCompletedByDefault ? "show" : "hide"})
                </option>
                <option value="show">Always show</option>
                <option value="hide">Hide completed</option>
              </select>
            </div>
          </div>

          <div className="mt-4 flex items-center justify-between">
            <div className="space-y-1">
              <Label htmlFor="default-completed">Family default completed view</Label>
              <input
                id="default-completed"
                type="checkbox"
                checked={preferences.showCompletedByDefault}
                onChange={(event) =>
                  updatePreferences.mutate({ showCompletedByDefault: event.target.checked })
                }
              />
            </div>
            <Button
              variant="outline"
              onClick={() => clearCompleted.mutate()}
              disabled={!list.items.some((item) => item.completed)}
            >
              Remove all completed
            </Button>
          </div>
        </div>

        {sections.length === 0 ? (
          <div className="rounded-2xl bg-card p-6 text-center shadow-sm">
            <h3 className="text-lg font-semibold text-foreground">No items yet</h3>
            <p className="mt-2 text-sm text-muted-foreground">
              Add the first item to get this list started.
            </p>
          </div>
        ) : (
          sections.map((section) => (
            <div key={section.id} className="space-y-2">
              {section.title && (
                <h3 className="px-1 text-sm font-semibold uppercase tracking-wide text-muted-foreground">
                  {section.title}
                </h3>
              )}
              {section.items.map((item) => (
                <ListItemRow
                  key={item.id}
                  item={item}
                  onToggle={(completed) =>
                    updateItem.mutate({
                      itemId: item.id,
                      request: {
                        text: item.text,
                        completed,
                        categoryId: item.categoryId,
                      },
                    })
                  }
                  onEdit={() => setItemSheet({ mode: "edit", itemId: item.id })}
                  onDelete={() => deleteItem.mutate(item.id)}
                />
              ))}
            </div>
          ))
        )}

        <ListItemSheet
          open={itemSheet !== null}
          mode={itemSheet?.mode ?? "create"}
          list={list}
          item={list.items.find((item) => item.id === itemSheet?.itemId) ?? null}
          onOpenChange={(open) => !open && setItemSheet(null)}
        />
      </div>
    </div>
  );
}
```

- [ ] **Step 4: Re-run the frontend component tests**

Run:

```bash
cd frontend
npm run test -- --run src/components/lists/build-list-sections.test.ts src/components/lists-view.test.tsx
```

Expected: PASS for grouped/flat section building, completed visibility overrides, and clear-completed UI behavior.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/components/lists/list-item-sheet.tsx \
  src/components/lists/list-detail-view.tsx \
  src/components/lists/list-item-row.tsx \
  src/components/lists/build-list-sections.ts \
  src/components/lists/build-list-sections.test.ts \
  src/components/lists-view.tsx \
  src/components/lists-view.test.tsx
git commit -m "feat(lists): add list detail interactions"
```

## Task 7: Verify The Frontend Against The Released Backend

**Files:**
- Create: `frontend/e2e/mobile-lists.spec.ts`

- [ ] **Step 1: Write the mobile E2E that covers the real grocery-list flow**

```ts
import { expect, test } from "@playwright/test";
import { registerFamily, seedBrowserAuth } from "./helpers/api-helpers";
import { clearStorage, waitForHydration } from "./helpers/test-helpers";

test.describe("Mobile Lists", () => {
  test.beforeEach(async ({ page, isMobile }) => {
    test.skip(!isMobile, "Mobile-only tests");
    await page.goto("/");
    await clearStorage(page);
  });

  test("creates a grocery list, toggles categories, completes an item, and clears completed", async ({
    page,
    request,
  }) => {
    const registration = await registerFamily(request, {
      familyName: "Lists E2E Family",
      members: [{ name: "Alice", color: "coral" }],
    });
    await seedBrowserAuth(page, registration);

    await page.reload();
    await waitForHydration(page);

    const nav = page.getByRole("navigation", { name: /primary/i });
    await nav.getByRole("button", { name: "Lists" }).click();

    await page.getByRole("button", { name: "New List" }).click();
    await page.getByLabel("List name").fill("Trader Joe's Run");
    await page.getByRole("radio", { name: "Grocery" }).click();
    await page.getByRole("button", { name: "Create list" }).click();

    await expect(page.getByRole("heading", { name: "Trader Joe's Run" })).toBeVisible();
    await expect(page.getByText("Produce")).toBeVisible();

    await page.getByRole("button", { name: "Add item" }).click();
    await page.getByLabel("Item text").fill("Bananas");
    await page.getByRole("combobox", { name: "Category" }).click();
    await page.getByRole("option", { name: "Produce" }).click();
    await page.getByRole("button", { name: "Save item" }).click();

    await expect(page.getByText("Bananas")).toBeVisible();

    await page.getByLabel("Categories").selectOption("flat");
    await expect(page.getByText("Produce")).toBeHidden();

    await page.getByRole("button", { name: /bananas/i }).click();
    await expect(page.getByText("Bananas")).toHaveCSS("text-decoration-line", "line-through");

    await page.getByRole("button", { name: "Remove all completed" }).click();
    await expect(page.getByText("Bananas")).toBeHidden();
  });
});
```

- [ ] **Step 2: Run the focused frontend unit tests and build before E2E**

Run:

```bash
cd frontend
npm run test -- --run \
  src/api/hooks/use-lists.test.tsx \
  src/lib/validations/lists.test.ts \
  src/components/lists/build-list-sections.test.ts \
  src/components/lists-view.test.tsx
npm run build
```

Expected: PASS for targeted unit tests, then PASS for the production build.

- [ ] **Step 3: After the backend release is published, run the mobile lists E2E**

Run:

```bash
cd frontend
npm run test:e2e -- e2e/mobile-lists.spec.ts --project=mobile-chrome
```

Expected: PASS for the real mobile grocery-list flow against the released backend.

- [ ] **Step 4: Commit**

```bash
cd frontend
git add e2e/mobile-lists.spec.ts
git commit -m "test(lists): add mobile lists e2e coverage"
```

## Spec Coverage Check

- Persisted family-scoped lists: Tasks 1-3
- More than one active list / hub view: Tasks 4-6
- Kind-gated seeded categories: Tasks 1-3 and 6
- Category show/hide modes: Task 6
- Family-wide completed default plus per-list override: Tasks 2, 4, and 6
- Completed styling and clear-completed action: Tasks 2, 6, and 7
- No assignees / no due dates / no chore drift: enforced by Tasks 2 and 6
- Empty states for `no lists yet` and `list has no items`: Tasks 5 and 6

## Delivery Notes

- Root follow-up after this plan is approved:
  - Open one BE execution issue in `joe-bor/family-hub-api`
  - Open one FE execution issue in `joe-bor/FamilyHub`
  - Put `Story:`, `Spec:`, and `Plan:` links at the top of each issue body
  - Copy the non-negotiable execution contract from the spec into each issue body
- Update `docs/product/backlog/module-foundations/lists-simple-shared-checklists.md` only when execution issues and PRs actually exist, or when the story state meaningfully changes.
- Because this root workspace is orchestration-only, implementation should happen from the owning delivery repos, not from `/Users/joe.bor/code/family-hub`.
