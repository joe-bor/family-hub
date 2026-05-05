# Chores Core Loop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the first real `Chores` module as a family-owned, mobile-first assignee board with persisted create / complete / delete behavior across backend and frontend.

**Architecture:** Implement the backend contract first in `backend/family-hub-api/`, release it, then wire the frontend `Chores` board in `frontend/` against that released API. Backend owns persistence, family-scoped authorization, and the chore DTO contract; frontend owns urgency sorting, assignee-lane presentation, optimistic UX, and the mobile create flow.

**Tech Stack:** Spring Boot 4, Java 21, JPA/Hibernate, Maven, React 19, TypeScript, TanStack Query, React Hook Form, Zod, Vitest, MSW, Playwright.

**Spec:** `docs/superpowers/specs/2026-05-05-chores-core-loop-design.md`

---

## Execution Order

1. Backend contract and persistence
2. Backend tests + release
3. Frontend API/types/hooks against the released backend contract
4. Frontend UI and verification

Do not start FE work that depends on live `/api/chores` behavior until the backend release exists. Per root shipping rules, FE should treat only published BE releases as stable contracts.

## File Structure

### Root docs / orchestration

- Modify: `docs/product/backlog/module-foundations/chores-core-loop.md`
- Create: `docs/superpowers/plans/2026-05-05-chores-core-loop.md`

### Backend (`backend/family-hub-api/`)

Expected create/modify set:

- Create: `src/main/java/com/familyhub/demo/model/Chore.java`
- Create: `src/main/java/com/familyhub/demo/dto/CreateChoreRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/UpdateChoreRequest.java`
- Create: `src/main/java/com/familyhub/demo/dto/ChoreResponse.java`
- Create: `src/main/java/com/familyhub/demo/mapper/ChoreMapper.java`
- Create: `src/main/java/com/familyhub/demo/repository/ChoreRepository.java`
- Create: `src/main/java/com/familyhub/demo/service/ChoreService.java`
- Create: `src/main/java/com/familyhub/demo/controller/ChoreController.java`
- Modify: `src/test/java/com/familyhub/demo/TestDataFactory.java`
- Create: `src/test/java/com/familyhub/demo/controller/ChoreControllerTest.java`
- Create: `src/test/java/com/familyhub/demo/service/ChoreServiceTest.java`
- Create: `src/test/java/com/familyhub/demo/integration/ChoreIntegrationTest.java`

### Frontend (`frontend/`)

Expected create/modify set:

- Modify: `src/lib/types/chores.ts`
- Modify: `src/lib/types/index.ts`
- Create: `src/api/services/chores.service.ts`
- Modify: `src/api/services/index.ts`
- Create: `src/api/hooks/use-chores.ts`
- Modify: `src/api/hooks/index.ts`
- Modify: `src/api/index.ts`
- Create: `src/lib/validations/chores.ts`
- Modify: `src/lib/validations/index.ts`
- Modify: `src/test/mocks/handlers.ts`
- Modify: `src/test/mocks/server.ts`
- Delete: `src/stores/chores-store.ts`
- Modify: `src/stores/index.ts`
- Modify: `src/lib/calendar-data.ts`
- Create: `src/components/ui/mobile-sheet.tsx`
- Modify: `src/components/calendar/components/mobile-event-sheet.tsx`
- Create: `src/components/chores/chore-form.tsx`
- Create: `src/components/chores/chore-form-sheet.tsx`
- Create: `src/components/chores/chore-row.tsx`
- Create: `src/components/chores/chore-lane.tsx`
- Modify: `src/components/chores-view.tsx`
- Create: `src/api/hooks/use-chores.test.tsx`
- Create: `src/lib/validations/chores.test.ts`
- Create: `src/components/chores/chore-form.test.tsx`
- Create: `src/components/chores-view.test.tsx`
- Create: `e2e/mobile-chores.spec.ts`

This split keeps server state in `@/api`, UI state in components, and the `Chores` surface small enough to evolve without bloating `src/components/chores-view.tsx`.

## Task 1: Define The Backend Chore Contract

**Files:**
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/CreateChoreRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/UpdateChoreRequest.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/dto/ChoreResponse.java`
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/controller/ChoreControllerTest.java`
- Modify: `backend/family-hub-api/src/test/java/com/familyhub/demo/TestDataFactory.java`

- [ ] **Step 1: Write the failing controller tests for the chore endpoints**

```java
@WebMvcTest(ChoreController.class)
@Import({SecurityConfig.class, JwtAuthenticationFilter.class, JwtAuthenticationEntryPoint.class})
@ActiveProfiles("test")
class ChoreControllerTest {

    @Autowired
    MockMvc mockMvc;

    @MockitoBean
    JwtService jwtService;

    @MockitoBean
    FamilyService familyService;

    @MockitoBean
    ChoreService choreService;

    @Test
    @WithMockFamily
    void getChores_returns200() throws Exception {
        given(choreService.getChores(any(Family.class)))
                .willReturn(List.of(sampleChoreResponse()));

        mockMvc.perform(get("/api/chores"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data[0].title").value("🗑️ Take out trash"))
                .andExpect(jsonPath("$.data[0].assignedToMemberId").value(MEMBER_ID.toString()));
    }

    @Test
    @WithMockFamily
    void createChore_returns201WithLocationHeader() throws Exception {
        given(choreService.createChore(any(), any(Family.class)))
                .willReturn(sampleChoreResponse());

        mockMvc.perform(post("/api/chores")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(CREATE_CHORE_JSON))
                .andExpect(status().isCreated())
                .andExpect(header().string("Location", "/api/chores/" + CHORE_ID))
                .andExpect(jsonPath("$.message").value("Chore created successfully"));
    }

    @Test
    @WithMockFamily
    void updateChore_emptyBody_returns400() throws Exception {
        mockMvc.perform(patch("/api/chores/{id}", CHORE_ID)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{}"))
                .andExpect(status().isBadRequest());
    }

    @Test
    @WithMockFamily
    void deleteChore_returns204() throws Exception {
        willDoNothing().given(choreService).deleteChore(eq(CHORE_ID), any(Family.class));

        mockMvc.perform(delete("/api/chores/{id}", CHORE_ID))
                .andExpect(status().isNoContent());
    }
}
```

- [ ] **Step 2: Add chore test fixtures to `TestDataFactory`**

```java
public static final UUID CHORE_ID = UUID.fromString("00000000-0000-0000-0000-000000000004");

public static CreateChoreRequest createChoreRequest(UUID memberId) {
    return new CreateChoreRequest(
            "🗑️ Take out trash",
            memberId,
            LocalDate.of(2026, 5, 5)
    );
}

public static Chore createChore(Family family, FamilyMember member) {
    Chore chore = new Chore();
    chore.setId(CHORE_ID);
    chore.setFamily(family);
    chore.setAssignedToMember(member);
    chore.setTitle("🗑️ Take out trash");
    chore.setDueDate(LocalDate.of(2026, 5, 5));
    chore.setCompleted(false);
    chore.setCompletedAt(null);
    return chore;
}

public static Chore createCompletedChore(Family family, FamilyMember member) {
    Chore chore = createChore(family, member);
    chore.setCompleted(true);
    chore.setCompletedAt(LocalDateTime.of(2026, 5, 5, 10, 0));
    return chore;
}

public static ChoreResponse sampleChoreResponse() {
    return new ChoreResponse(
            CHORE_ID,
            "🗑️ Take out trash",
            MEMBER_ID,
            LocalDate.of(2026, 5, 5),
            false,
            null,
            LocalDateTime.of(2026, 5, 5, 9, 0),
            LocalDateTime.of(2026, 5, 5, 9, 0)
    );
}
```

- [ ] **Step 3: Add the DTO records that make the tests compile**

```java
public record CreateChoreRequest(
        @NotBlank
        @Size(max = 100, message = "Chore title must be 100 characters or less")
        String title,

        @NotNull
        UUID assignedToMemberId,

        @JsonFormat(pattern = "yyyy-MM-dd")
        LocalDate dueDate
) {}

public record UpdateChoreRequest(
        @NotNull
        Boolean completed
) {}

public record ChoreResponse(
        UUID id,
        String title,
        UUID assignedToMemberId,
        @JsonFormat(pattern = "yyyy-MM-dd")
        LocalDate dueDate,
        boolean completed,
        LocalDateTime completedAt,
        LocalDateTime createdAt,
        LocalDateTime updatedAt
) {}
```

- [ ] **Step 4: Run the controller test to verify the next failure is the missing controller/service layer**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=ChoreControllerTest test
```

Expected: test compilation reaches the new DTOs, then fails because `ChoreController` / `ChoreService` do not exist yet.

- [ ] **Step 5: Commit**

```bash
cd backend/family-hub-api
git add src/main/java/com/familyhub/demo/dto/CreateChoreRequest.java \
  src/main/java/com/familyhub/demo/dto/UpdateChoreRequest.java \
  src/main/java/com/familyhub/demo/dto/ChoreResponse.java \
  src/test/java/com/familyhub/demo/controller/ChoreControllerTest.java \
  src/test/java/com/familyhub/demo/TestDataFactory.java
git commit -m "feat(chores): add chore API contract tests"
```

## Task 2: Implement Backend Chore Persistence And Endpoints

**Files:**
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/model/Chore.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/mapper/ChoreMapper.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/repository/ChoreRepository.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/service/ChoreService.java`
- Create: `backend/family-hub-api/src/main/java/com/familyhub/demo/controller/ChoreController.java`
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/service/ChoreServiceTest.java`

- [ ] **Step 1: Write the failing service tests for family scoping, completion toggles, and deletes**

```java
class ChoreServiceTest {

    @Mock
    private ChoreRepository choreRepository;

    @Mock
    private FamilyMemberRepository familyMemberRepository;

    @InjectMocks
    private ChoreService choreService;

    @Test
    void updateChore_completedTrue_setsCompletedAt() {
        Family family = createFamily();
        FamilyMember member = createFamilyMember(family);
        Chore chore = createChore(family, member);

        when(choreRepository.findByFamilyAndId(family, CHORE_ID)).thenReturn(Optional.of(chore));

        choreService.updateChore(
                CHORE_ID,
                new UpdateChoreRequest(true),
                family
        );

        assertThat(chore.isCompleted()).isTrue();
        assertThat(chore.getCompletedAt()).isNotNull();
    }

    @Test
    void updateChore_completedFalse_clearsCompletedAt() {
        Family family = createFamily();
        FamilyMember member = createFamilyMember(family);
        Chore chore = createCompletedChore(family, member);

        when(choreRepository.findByFamilyAndId(family, CHORE_ID)).thenReturn(Optional.of(chore));

        choreService.updateChore(
                CHORE_ID,
                new UpdateChoreRequest(false),
                family
        );

        assertThat(chore.isCompleted()).isFalse();
        assertThat(chore.getCompletedAt()).isNull();
    }
}
```

- [ ] **Step 2: Create the entity, mapper, and repository**

```java
@Entity
@Getter
@Setter
public class Chore {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @ManyToOne
    @JoinColumn(name = "family_id", nullable = false)
    private Family family;

    @ManyToOne
    @JoinColumn(name = "assigned_to_member_id", nullable = false)
    private FamilyMember assignedToMember;

    @Column(nullable = false, length = 100)
    private String title;

    private LocalDate dueDate;

    @Column(nullable = false)
    private boolean completed;

    private LocalDateTime completedAt;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}

public interface ChoreRepository extends JpaRepository<Chore, UUID> {
    @Query("select c from Chore c join fetch c.assignedToMember where c.family = :family")
    List<Chore> findByFamilyWithAssignee(@Param("family") Family family);

    Optional<Chore> findByFamilyAndId(Family family, UUID id);
}
```

- [ ] **Step 3: Implement the mapper, service, and controller**

```java
public class ChoreMapper {
    private ChoreMapper() {}

    public static ChoreResponse toDto(Chore chore) {
        return new ChoreResponse(
                chore.getId(),
                chore.getTitle(),
                chore.getAssignedToMember().getId(),
                chore.getDueDate(),
                chore.isCompleted(),
                chore.getCompletedAt(),
                chore.getCreatedAt(),
                chore.getUpdatedAt()
        );
    }
}

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ChoreService {
    private final ChoreRepository choreRepository;
    private final FamilyMemberRepository familyMemberRepository;

    public List<ChoreResponse> getChores(Family family) {
        return choreRepository.findByFamilyWithAssignee(family)
                .stream()
                .map(ChoreMapper::toDto)
                .toList();
    }

    @Transactional
    public ChoreResponse createChore(CreateChoreRequest request, Family family) {
        FamilyMember member = resolveFamilyMember(family, request.assignedToMemberId());

        Chore chore = new Chore();
        chore.setFamily(family);
        chore.setAssignedToMember(member);
        chore.setTitle(request.title().trim());
        chore.setDueDate(request.dueDate());
        chore.setCompleted(false);
        chore.setCompletedAt(null);

        return ChoreMapper.toDto(choreRepository.save(chore));
    }

    @Transactional
    public ChoreResponse updateChore(UUID id, UpdateChoreRequest request, Family family) {
        Chore chore = choreRepository.findByFamilyAndId(family, id)
                .orElseThrow(() -> new ResourceNotFoundException("Chore", id));

        chore.setCompleted(request.completed());
        chore.setCompletedAt(Boolean.TRUE.equals(request.completed()) ? LocalDateTime.now() : null);

        return ChoreMapper.toDto(choreRepository.save(chore));
    }

    @Transactional
    public void deleteChore(UUID id, Family family) {
        Chore chore = choreRepository.findByFamilyAndId(family, id)
                .orElseThrow(() -> new ResourceNotFoundException("Chore", id));

        choreRepository.delete(chore);
    }

    private FamilyMember resolveFamilyMember(Family family, UUID memberId) {
        FamilyMember member = familyMemberRepository.findById(memberId)
                .orElseThrow(() -> new ResourceNotFoundException("Family Member", memberId));

        if (!member.getFamily().getId().equals(family.getId())) {
            throw new AccessDeniedException("Unauthorized");
        }

        return member;
    }
}

@RestController
@RequestMapping("/api/chores")
@RequiredArgsConstructor
public class ChoreController {
    private final ChoreService choreService;

    @GetMapping
    public ResponseEntity<ApiResponse<List<ChoreResponse>>> getChores(@AuthenticationPrincipal Family family) {
        return ResponseEntity.ok(new ApiResponse<>(choreService.getChores(family), ""));
    }

    @PostMapping
    public ResponseEntity<ApiResponse<ChoreResponse>> createChore(
            @Valid @RequestBody CreateChoreRequest request,
            @AuthenticationPrincipal Family family
    ) {
        ChoreResponse response = choreService.createChore(request, family);
        return ResponseEntity.created(URI.create("/api/chores/" + response.id()))
                .body(new ApiResponse<>(response, "Chore created successfully"));
    }

    @PatchMapping("/{id}")
    public ResponseEntity<ApiResponse<ChoreResponse>> updateChore(
            @PathVariable UUID id,
            @Valid @RequestBody UpdateChoreRequest request,
            @AuthenticationPrincipal Family family
    ) {
        return ResponseEntity.ok(new ApiResponse<>(choreService.updateChore(id, request, family), "Chore updated successfully"));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteChore(@PathVariable UUID id, @AuthenticationPrincipal Family family) {
        choreService.deleteChore(id, family);
        return ResponseEntity.noContent().build();
    }
}
```

- [ ] **Step 4: Run the controller and service tests to make them pass**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=ChoreControllerTest,ChoreServiceTest test
```

Expected: PASS for the new controller/service test classes.

- [ ] **Step 5: Commit**

```bash
cd backend/family-hub-api
git add src/main/java/com/familyhub/demo/model/Chore.java \
  src/main/java/com/familyhub/demo/mapper/ChoreMapper.java \
  src/main/java/com/familyhub/demo/repository/ChoreRepository.java \
  src/main/java/com/familyhub/demo/service/ChoreService.java \
  src/main/java/com/familyhub/demo/controller/ChoreController.java \
  src/test/java/com/familyhub/demo/service/ChoreServiceTest.java
git commit -m "feat(chores): implement family chore endpoints"
```

## Task 3: Verify Backend Persistence And Release The Contract

**Files:**
- Create: `backend/family-hub-api/src/test/java/com/familyhub/demo/integration/ChoreIntegrationTest.java`

- [ ] **Step 1: Write the failing integration test for the create → complete → delete lifecycle**

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
class ChoreIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @WithMockFamily
    void createToggleDeleteChore_roundTripsThroughTheApi() throws Exception {
        String body = """
                {
                  "title": "🪥 Brush teeth",
                  "assignedToMemberId": "00000000-0000-0000-0000-000000000002",
                  "dueDate": "2026-05-05"
                }
                """;

        String location = mockMvc.perform(post("/api/chores")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(body))
                .andExpect(status().isCreated())
                .andReturn()
                .getResponse()
                .getHeader("Location");

        String id = location.substring(location.lastIndexOf('/') + 1);

        mockMvc.perform(patch("/api/chores/{id}", id)
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"completed\":true}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.completed").value(true));

        mockMvc.perform(delete("/api/chores/{id}", id))
                .andExpect(status().isNoContent());
    }
}
```

- [ ] **Step 2: Run the focused integration test**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=ChoreIntegrationTest test
```

Expected: PASS with a green create / patch / delete lifecycle.

- [ ] **Step 3: Run the backend verification suite for the new story**

Run:

```bash
cd backend/family-hub-api
./mvnw -Dtest=ChoreControllerTest,ChoreServiceTest,ChoreIntegrationTest test
```

Expected: PASS for all three new test classes.

- [ ] **Step 4: Merge with `feat:` commits intact so release-please can publish the backend contract**

```bash
cd backend/family-hub-api
git log --oneline --decorate -n 5
```

Expected: the chore work is represented by merge-visible `feat(chores): ...` commits so release-please will cut a new backend release after merge to `main`.

- [ ] **Step 5: Commit**

```bash
cd backend/family-hub-api
git add src/test/java/com/familyhub/demo/integration/ChoreIntegrationTest.java
git commit -m "feat(chores): add chore integration coverage"
```

## Task 4: Add Frontend Chore Types, Service Hooks, And Test Mocks

**Files:**
- Modify: `frontend/src/lib/types/chores.ts`
- Modify: `frontend/src/lib/types/index.ts`
- Create: `frontend/src/api/services/chores.service.ts`
- Modify: `frontend/src/api/services/index.ts`
- Create: `frontend/src/api/hooks/use-chores.ts`
- Modify: `frontend/src/api/hooks/index.ts`
- Modify: `frontend/src/api/index.ts`
- Modify: `frontend/src/test/mocks/handlers.ts`
- Modify: `frontend/src/test/mocks/server.ts`
- Delete: `frontend/src/stores/chores-store.ts`
- Modify: `frontend/src/stores/index.ts`
- Modify: `frontend/src/lib/calendar-data.ts`
- Create: `frontend/src/api/hooks/use-chores.test.tsx`

- [ ] **Step 1: Write the failing hook tests for fetch, create, toggle, and delete**

```tsx
describe("useChores hooks", () => {
  setupMswServer();

  it("returns chores from the API", async () => {
    seedMockFamily({
      id: "family-1",
      name: "Test Family",
      members: [{ id: "member-1", name: "Leo", color: "coral" }],
    });
    seedMockChores([
      {
        id: "chore-1",
        title: "🗑️ Take out trash",
        assignedToMemberId: "member-1",
        dueDate: "2026-05-05",
        completed: false,
        completedAt: null,
        createdAt: "2026-05-05T09:00:00",
        updatedAt: "2026-05-05T09:00:00",
      },
    ]);

    const queryClient = createTestQueryClient();
    const wrapper = ({ children }: { children: React.ReactNode }) => (
      <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
    );

    const { result } = renderHook(() => useChores(), { wrapper });

    await waitFor(() => {
      expect(result.current.data?.data[0].title).toBe("🗑️ Take out trash");
    });
  });

  it("optimistically removes a deleted chore", async () => {
    seedMockFamily({
      id: "family-1",
      name: "Test Family",
      members: [{ id: "member-1", name: "Leo", color: "coral" }],
    });
    seedMockChores([
      {
        id: "chore-1",
        title: "🗑️ Take out trash",
        assignedToMemberId: "member-1",
        dueDate: "2026-05-05",
        completed: false,
        completedAt: null,
        createdAt: "2026-05-05T09:00:00",
        updatedAt: "2026-05-05T09:00:00",
      },
    ]);

    const queryClient = createTestQueryClient();
    queryClient.setQueryData(choresKeys.list(), {
      data: [
        {
          id: "chore-1",
          title: "🗑️ Take out trash",
          assignedToMemberId: "member-1",
          dueDate: parseLocalDate("2026-05-05"),
          completed: false,
          completedAt: null,
          createdAt: "2026-05-05T09:00:00",
          updatedAt: "2026-05-05T09:00:00",
        },
      ],
    });

    const wrapper = ({ children }: { children: React.ReactNode }) => (
      <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
    );

    const { result } = renderHook(() => useDeleteChore(), { wrapper });
    result.current.mutate("chore-1");

    await waitFor(() => {
      const cached = queryClient.getQueryData<ApiResponse<Chore[]>>(choresKeys.list());
      expect(cached?.data).toEqual([]);
    });
  });
});
```

- [ ] **Step 2: Replace the placeholder type with real API/domain chore types**

```ts
export interface ChoreResponse {
  id: string;
  title: string;
  assignedToMemberId: string;
  dueDate: string | null;
  completed: boolean;
  completedAt: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface Chore {
  id: string;
  title: string;
  assignedToMemberId: string;
  dueDate?: Date;
  completed: boolean;
  completedAt: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface CreateChoreRequest {
  title: string;
  assignedToMemberId: string;
  dueDate?: string | null;
}

export interface UpdateChoreRequest {
  completed: boolean;
}
```

- [ ] **Step 3: Implement the chore service, query hooks, and MSW handlers**

```ts
function toChore(raw: ChoreResponse): Chore {
  return {
    ...raw,
    dueDate: raw.dueDate ? parseLocalDate(raw.dueDate) : undefined,
  };
}

function mapChoreResponse(response: ApiResponse<ChoreResponse>): ApiResponse<Chore> {
  return { ...response, data: toChore(response.data) };
}

function mapChoresResponse(
  response: ApiResponse<ChoreResponse[]>,
): ApiResponse<Chore[]> {
  return { ...response, data: response.data.map(toChore) };
}

export const choreService = {
  async getChores(): Promise<ApiResponse<Chore[]>> {
    return mapChoresResponse(
      await httpClient.get<ApiResponse<ChoreResponse[]>>("/chores"),
    );
  },

  async createChore(
    request: CreateChoreRequest,
  ): Promise<ApiResponse<Chore>> {
    return mapChoreResponse(
      await httpClient.post<ApiResponse<ChoreResponse>>("/chores", request),
    );
  },

  async updateChore(
    id: string,
    request: UpdateChoreRequest,
  ): Promise<ApiResponse<Chore>> {
    return mapChoreResponse(
      await httpClient.patch<ApiResponse<ChoreResponse>>(`/chores/${id}`, request),
    );
  },

  async deleteChore(id: string): Promise<void> {
    return httpClient.delete(`/chores/${id}`);
  },
};

export const choresKeys = {
  all: ["chores"] as const,
  list: () => [...choresKeys.all, "list"] as const,
};

export function useChores() {
  return useQuery({
    queryKey: choresKeys.list(),
    queryFn: choreService.getChores,
    staleTime: 5 * 60 * 1000,
  });
}

export function useCreateChore() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: choreService.createChore,
    onSuccess: (response) => {
      queryClient.setQueryData<ApiResponse<Chore[]>>(choresKeys.list(), (old) =>
        old ? { ...old, data: [...old.data, response.data] } : { data: [response.data] },
      );
    },
  });
}

export function useUpdateChore() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, request }: { id: string; request: UpdateChoreRequest }) =>
      choreService.updateChore(id, request),
    onMutate: async ({ id, request }) => {
      await queryClient.cancelQueries({ queryKey: choresKeys.list() });
      const previous = queryClient.getQueryData<ApiResponse<Chore[]>>(choresKeys.list());

      queryClient.setQueryData<ApiResponse<Chore[]>>(choresKeys.list(), (old) =>
        old
          ? {
              ...old,
              data: old.data.map((chore) =>
                chore.id === id
                  ? {
                      ...chore,
                      completed: request.completed,
                      completedAt: request.completed ? new Date().toISOString() : null,
                    }
                  : chore,
              ),
            }
          : old,
      );

      return { previous };
    },
  });
}

export function useDeleteChore() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: choreService.deleteChore,
    onMutate: async (id) => {
      await queryClient.cancelQueries({ queryKey: choresKeys.list() });
      const previous = queryClient.getQueryData<ApiResponse<Chore[]>>(choresKeys.list());

      queryClient.setQueryData<ApiResponse<Chore[]>>(choresKeys.list(), (old) =>
        old ? { ...old, data: old.data.filter((chore) => chore.id !== id) } : old,
      );

      return { previous };
    },
  });
}

let mockChores: ChoreResponse[] = [];

export function seedMockChores(chores: ChoreResponse[]): void {
  mockChores = [...chores];
}
```

- [ ] **Step 4: Run the focused hook test suite**

Run:

```bash
cd frontend
npm run test -- --run src/api/hooks/use-chores.test.tsx
```

Expected: PASS for fetch/create/update/delete hook coverage with MSW-backed optimistic assertions.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/lib/types/chores.ts \
  src/lib/types/index.ts \
  src/api/services/chores.service.ts \
  src/api/services/index.ts \
  src/api/hooks/use-chores.ts \
  src/api/hooks/index.ts \
  src/api/index.ts \
  src/test/mocks/handlers.ts \
  src/test/mocks/server.ts \
  src/api/hooks/use-chores.test.tsx \
  src/stores/index.ts \
  src/lib/calendar-data.ts
git rm src/stores/chores-store.ts
git commit -m "feat(chores): add frontend chore data hooks"
```

## Task 5: Build The Chore Create Flow And Shared Mobile Sheet

**Files:**
- Create: `frontend/src/components/ui/mobile-sheet.tsx`
- Modify: `frontend/src/components/calendar/components/mobile-event-sheet.tsx`
- Create: `frontend/src/lib/validations/chores.ts`
- Modify: `frontend/src/lib/validations/index.ts`
- Create: `frontend/src/components/chores/chore-form.tsx`
- Create: `frontend/src/components/chores/chore-form-sheet.tsx`
- Create: `frontend/src/lib/validations/chores.test.ts`
- Create: `frontend/src/components/chores/chore-form.test.tsx`

- [ ] **Step 1: Write the failing validation and form tests**

```ts
describe("choreFormSchema", () => {
  it("requires title and assignee", () => {
    const result = choreFormSchema.safeParse({
      title: "",
      assignedToMemberId: "",
      dueDate: undefined,
    });

    expect(result.success).toBe(false);
    expect(result.error.issues.map((issue) => issue.path.join("."))).toEqual(
      expect.arrayContaining(["title", "assignedToMemberId"]),
    );
  });
});
```

```tsx
it("submits title, assignee, and due date", async () => {
  seedFamilyStore({
    members: [{ id: "member-1", name: "Leo", color: "coral" }],
  });

  const onSubmit = vi.fn();
  const { user } = renderWithUser(
    <ChoreForm
      defaultValues={{ assignedToMemberId: "member-1" }}
      onSubmit={onSubmit}
      onCancel={vi.fn()}
    />,
  );

  await user.type(screen.getByLabelText(/chore name/i), "🗑️ Take out trash");
  await user.click(screen.getByRole("button", { name: /leo/i }));
  await user.click(screen.getByRole("button", { name: /save chore/i }));

  await waitFor(() => {
    expect(onSubmit).toHaveBeenCalledWith(
      expect.objectContaining({
        title: "🗑️ Take out trash",
        assignedToMemberId: "member-1",
      }),
    );
  });
});
```

- [ ] **Step 2: Extract a reusable full-screen mobile sheet**

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
      className="fixed inset-0 z-50 flex flex-col bg-card motion-safe:animate-in motion-safe:slide-in-from-bottom motion-safe:duration-200"
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

- [ ] **Step 3: Implement the chore form schema and sheet**

```ts
export const choreFormSchema = z.object({
  title: z
    .string()
    .trim()
    .min(1, "Chore name is required")
    .max(100, "Chore name must be 100 characters or less"),
  assignedToMemberId: z.string().min(1, "Assignee is required"),
  dueDate: z.string().optional(),
});

export type ChoreFormData = z.infer<typeof choreFormSchema>;
```

```tsx
export function ChoreFormSheet({
  isOpen,
  onClose,
  onSubmit,
  isPending,
}: ChoreFormSheetProps) {
  return (
    <MobileSheet isOpen={isOpen} onClose={onClose} title="New Chore">
      <ChoreForm onSubmit={onSubmit} onCancel={onClose} isPending={isPending} />
    </MobileSheet>
  );
}
```

- [ ] **Step 4: Run the validation and form tests**

Run:

```bash
cd frontend
npm run test -- --run src/lib/validations/chores.test.ts src/components/chores/chore-form.test.tsx
```

Expected: PASS for schema validation and the create form submission flow.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/components/ui/mobile-sheet.tsx \
  src/components/calendar/components/mobile-event-sheet.tsx \
  src/lib/validations/chores.ts \
  src/lib/validations/index.ts \
  src/components/chores/chore-form.tsx \
  src/components/chores/chore-form-sheet.tsx \
  src/lib/validations/chores.test.ts \
  src/components/chores/chore-form.test.tsx
git commit -m "feat(chores): add mobile chore create flow"
```

## Task 6: Build The Assignee-Lane Board And Optimistic UX

**Files:**
- Create: `frontend/src/components/chores/chore-row.tsx`
- Create: `frontend/src/components/chores/chore-lane.tsx`
- Modify: `frontend/src/components/chores-view.tsx`
- Create: `frontend/src/components/chores-view.test.tsx`

- [ ] **Step 1: Write the failing view tests for grouping, urgency sort, completed-at-bottom, and all-caught-up**

```tsx
it("groups incomplete chores by assignee and sorts by urgency", async () => {
  seedFamilyStore({
    members: [
      { id: "leo", name: "Leo", color: "coral" },
      { id: "maya", name: "Maya", color: "teal" },
    ],
  });
  seedMockChores([
    createMockChore({ id: "1", title: "Future", assignedToMemberId: "leo", dueDate: "2026-05-10" }),
    createMockChore({ id: "2", title: "Overdue", assignedToMemberId: "leo", dueDate: "2026-05-01" }),
    createMockChore({ id: "3", title: "Today", assignedToMemberId: "leo", dueDate: "2026-05-05" }),
    createMockChore({ id: "4", title: "Done", assignedToMemberId: "leo", dueDate: "2026-05-02", completed: true, completedAt: "2026-05-05T08:00:00" }),
  ]);

  render(<ChoresView />);

  const rows = await screen.findAllByTestId(/chore-row-/);
  expect(rows.map((row) => row.textContent)).toEqual([
    expect.stringContaining("Overdue"),
    expect.stringContaining("Today"),
    expect.stringContaining("Future"),
    expect.stringContaining("Done"),
  ]);
});

it("shows the all caught up state when only completed chores remain", async () => {
  // seed only completed chores and assert banner + completed-only lanes
});
```

Replace `createMockChore(...)` above with a tiny local helper inside the test file:

```tsx
function createMockChore(
  overrides: Partial<ChoreResponse> & Pick<ChoreResponse, "id" | "title" | "assignedToMemberId">,
): ChoreResponse {
  return {
    id: overrides.id,
    title: overrides.title,
    assignedToMemberId: overrides.assignedToMemberId,
    dueDate: overrides.dueDate ?? null,
    completed: overrides.completed ?? false,
    completedAt: overrides.completedAt ?? null,
    createdAt: overrides.createdAt ?? "2026-05-05T09:00:00",
    updatedAt: overrides.updatedAt ?? "2026-05-05T09:00:00",
  };
}
```

- [ ] **Step 2: Implement row and lane components with visual progress and delete affordance**

```tsx
export function ChoreRow({
  chore,
  onToggleComplete,
  onDelete,
}: ChoreRowProps) {
  return (
    <div
      data-testid={`chore-row-${chore.id}`}
      className={cn(
        "flex items-center gap-3 rounded-xl border p-3",
        chore.completed
          ? "border-border bg-muted/40"
          : "border-transparent bg-card hover:border-border",
      )}
    >
      <button
        type="button"
        aria-label={chore.completed ? "Mark incomplete" : "Mark complete"}
        onClick={onToggleComplete}
        className="flex h-8 w-8 shrink-0 items-center justify-center rounded-full border-2"
      />
      <div className="min-w-0 flex-1">
        <p className={cn("text-sm font-semibold", chore.completed && "text-muted-foreground line-through")}>
          {chore.title}
        </p>
        {chore.dueDate && <p className="mt-1 text-xs text-muted-foreground">{formatDueDateLabel(chore.dueDate)}</p>}
      </div>
      <button type="button" aria-label={`Delete ${chore.title}`} onClick={onDelete}>
        <MoreHorizontal className="h-4 w-4" />
      </button>
    </div>
  );
}

export function ChoreLane({ member, chores, onToggleComplete, onDelete }: ChoreLaneProps) {
  const completedCount = chores.filter((chore) => chore.completed).length;
  const progressPercent = chores.length === 0 ? 0 : (completedCount / chores.length) * 100;

  return (
    <section className="overflow-hidden rounded-2xl border border-border bg-card shadow-sm">
      <header className={cn("px-5 py-4", colorMap[member.color].light)}>
        <div className="flex items-center gap-3">
          <MemberAvatar name={member.name} color={member.color} />
          <div className="min-w-0 flex-1">
            <h2 className="text-base font-semibold text-foreground">{member.name}</h2>
          </div>
        </div>
        <div className="mt-3 h-2 overflow-hidden rounded-full bg-white/40">
          <div className={cn("h-full rounded-full", colorMap[member.color].bg)} style={{ width: `${progressPercent}%` }} />
        </div>
      </header>
      <div className="space-y-2 p-4">
        {chores.map((chore) => (
          <ChoreRow
            key={chore.id}
            chore={chore}
            onToggleComplete={() => onToggleComplete(chore)}
            onDelete={() => onDelete(chore)}
          />
        ))}
      </div>
    </section>
  );
}
```

- [ ] **Step 3: Replace the local state view with query-backed optimistic mutations**

```tsx
export function ChoresView() {
  const [isCreateOpen, setCreateOpen] = useState(false);
  const members = useFamilyMembers();
  const { data, isLoading } = useChores();
  const createChore = useCreateChore();
  const updateChore = useUpdateChore();
  const deleteChore = useDeleteChore();

  const chores = useMemo(() => data?.data ?? [], [data]);
  const grouped = useMemo(() => {
    const today = formatLocalDate(new Date());
    return buildChoreLanes({ chores, members, today });
  }, [chores, members]);

  if (isLoading) return <div className="p-4">Loading chores…</div>;

  return (
    <>
      <div className="flex-1 overflow-y-auto p-4 sm:p-6">
        <div className="mb-6 flex items-center justify-between gap-3">
          <h1 className="text-[24px] leading-8 font-semibold text-foreground">
            Chores
          </h1>
          <Button onClick={() => setCreateOpen(true)}>+</Button>
        </div>

        {grouped.mode === "empty" && (
          <div className="py-20 text-center">
            <h2 className="text-lg font-semibold text-foreground">No chores yet</h2>
            <p className="mt-2 text-sm text-muted-foreground">
              Add the first chore to start the family board.
            </p>
          </div>
        )}

        {grouped.mode === "all-caught-up" && (
          <div className="mb-6 rounded-2xl border border-border bg-card p-4 shadow-sm">
            <h2 className="text-base font-semibold text-foreground">All caught up</h2>
            <p className="mt-1 text-sm text-muted-foreground">
              Everything is done for now. Completed chores stay visible below.
            </p>
          </div>
        )}

        <div className="space-y-4">
          {grouped.lanes.map((lane) => (
            <ChoreLane
              key={lane.member.id}
              member={lane.member}
              chores={lane.chores}
              onToggleComplete={(chore) =>
                updateChore.mutate({
                  id: chore.id,
                  request: { completed: !chore.completed },
                })
              }
              onDelete={(chore) => deleteChore.mutate(chore.id)}
            />
          ))}
        </div>
      </div>
      <ChoreFormSheet
        isOpen={isCreateOpen}
        onClose={() => setCreateOpen(false)}
        isPending={createChore.isPending}
        onSubmit={(values) =>
          createChore.mutate({
            title: values.title,
            assignedToMemberId: values.assignedToMemberId,
            dueDate: values.dueDate ?? null,
          })
        }
      />
    </>
  );
}
```

Include the helper in `src/components/chores-view.tsx` so the sort and state rules are explicit in one place:

```tsx
function buildChoreLanes({
  chores,
  members,
  today,
}: {
  chores: Chore[];
  members: FamilyMember[];
  today: string;
}) {
  const withMembers = members.map((member) => {
    const memberChores = chores.filter((chore) => chore.assignedToMemberId === member.id);
    const incomplete = memberChores
      .filter((chore) => !chore.completed)
      .sort((a, b) => compareChoreUrgency(a, b, today));
    const completed = memberChores.filter((chore) => chore.completed);

    return {
      member,
      chores: [...incomplete, ...completed],
      hasIncomplete: incomplete.length > 0,
      hasAny: memberChores.length > 0,
    };
  });

  const activeLanes = withMembers.filter((lane) => lane.hasIncomplete);
  if (activeLanes.length > 0) {
    return { mode: "active" as const, lanes: activeLanes };
  }

  const completedOnlyLanes = withMembers.filter((lane) => lane.hasAny);
  if (completedOnlyLanes.length > 0) {
    return { mode: "all-caught-up" as const, lanes: completedOnlyLanes };
  }

  return { mode: "empty" as const, lanes: [] };
}

function compareChoreUrgency(a: Chore, b: Chore, today: string) {
  const rank = (chore: Chore) => {
    if (!chore.dueDate) return 3;

    const due = formatLocalDate(chore.dueDate);
    if (due < today) return 0;
    if (due === today) return 1;
    return 2;
  };

  return rank(a) - rank(b);
}
```

- [ ] **Step 4: Run the focused board tests**

Run:

```bash
cd frontend
npm run test -- --run src/components/chores-view.test.tsx
```

Expected: PASS for grouping, urgency sorting, completed rows at the bottom, delete affordance, and all-caught-up behavior.

- [ ] **Step 5: Commit**

```bash
cd frontend
git add src/components/chores/chore-row.tsx \
  src/components/chores/chore-lane.tsx \
  src/components/chores-view.tsx \
  src/components/chores-view.test.tsx
git commit -m "feat(chores): add assignee lane board"
```

## Task 7: Verify Frontend Against The Released Backend Contract

**Files:**
- Create: `frontend/e2e/mobile-chores.spec.ts`

- [ ] **Step 1: Write the failing mobile E2E for create, complete, and delete**

```ts
test("family can create, complete, and delete a chore on mobile", async ({
  page,
  request,
}) => {
  const registration = await registerFamily(request, {
    familyName: "Chore Test Family",
    username: `chores-${Date.now()}`,
    password: "password123",
    members: [
      { name: "Leo", color: "coral" },
      { name: "Maya", color: "teal" },
    ],
  });

  await seedBrowserAuth(page, registration);
  await page.goto("/");

  await page.getByRole("button", { name: /^chores$/i }).click();
  await page.getByRole("button", { name: /\+/i }).click();
  await page.getByLabel(/chore name/i).fill("🗑️ Take out trash");
  await page.getByRole("button", { name: /leo/i }).click();
  await page.getByRole("button", { name: /save chore/i }).click();

  await expect(page.getByText("🗑️ Take out trash")).toBeVisible();
  await page.getByLabel(/mark complete/i).click();
  await expect(page.getByText("🗑️ Take out trash")).toHaveClass(/line-through/);

  await page.getByLabel(/delete 🗑️ take out trash/i).click();
  await page.getByRole("button", { name: /confirm delete/i }).click();
  await expect(page.getByText("🗑️ Take out trash")).not.toBeVisible();
});
```

- [ ] **Step 2: Run FE verification only after the backend release containing `/api/chores` is published**

Run:

```bash
cd frontend
npm run test -- --run \
  src/api/hooks/use-chores.test.tsx \
  src/lib/validations/chores.test.ts \
  src/components/chores/chore-form.test.tsx \
  src/components/chores-view.test.tsx
```

Expected: PASS for the focused unit/integration suite.

- [ ] **Step 3: Run build and mobile E2E against the released backend contract**

Run:

```bash
cd frontend
npm run build
npm run test:e2e -- mobile-bottom-nav.spec.ts mobile-chores.spec.ts
```

Expected: build succeeds, then mobile chores create / complete / delete passes against the released backend image tag.

- [ ] **Step 4: Commit**

```bash
cd frontend
git add e2e/mobile-chores.spec.ts
git commit -m "feat(chores): add mobile chores e2e coverage"
```

## Task 8: Open Execution Issues And Update Root Story Links

**Files:**
- Modify: `docs/product/backlog/module-foundations/chores-core-loop.md`

- [ ] **Step 1: Open the backend execution issue from this plan**

Use this issue body header:

```md
Story: https://github.com/joe-bor/family-hub/blob/main/docs/product/backlog/module-foundations/chores-core-loop.md
Spec: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/specs/2026-05-05-chores-core-loop-design.md
Plan: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/plans/2026-05-05-chores-core-loop.md

## Execution contract

- Implement persisted family-scoped chores in `family-hub-api`.
- Chores belong to the family account and are assigned to one family member.
- Do not add user identities or `completedBy`.
- Ship GET / POST / PATCH / DELETE `/api/chores`.
- FE will consume only a released backend contract.
```

- [ ] **Step 2: Open the frontend execution issue from this plan**

Use this issue body header:

```md
Story: https://github.com/joe-bor/family-hub/blob/main/docs/product/backlog/module-foundations/chores-core-loop.md
Spec: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/specs/2026-05-05-chores-core-loop-design.md
Plan: https://github.com/joe-bor/family-hub/blob/main/docs/superpowers/plans/2026-05-05-chores-core-loop.md

## Execution contract

- Replace placeholder chores with the released backend chore API.
- Build a mobile-first assignee-lane board with progress bars.
- Use the full-screen mobile sheet create flow.
- Keep completed chores visible at the bottom of each lane.
- Do not reintroduce sample chores in the shipping path.
```

- [ ] **Step 3: Record the FE and BE issue numbers in the story frontmatter**

```yaml
issues:
  - BE #<backend-issue-number>
  - FE #<frontend-issue-number>
```

- [ ] **Step 4: Commit**

```bash
cd /Users/joe.bor/code/family-hub
git add docs/product/backlog/module-foundations/chores-core-loop.md
git commit -m "docs: link chores execution issues"
```

## Self-Review Checklist

- Spec coverage:
  - family-owned chore model: Tasks 1-3
  - assignee-lane board: Tasks 5-6
  - create / complete / delete loop: Tasks 2, 4, 6, 7
  - dynamic progress bar: Task 6
  - no sample chores in shipping path: Tasks 4 and 6
  - FE waits for released BE contract: Tasks 3 and 7
- Placeholder scan:
  - no `TBD`, `TODO`, or “implement later” placeholders remain
  - each code-changing task includes representative code and exact file paths
- Type consistency:
  - backend uses `assignedToMemberId`
  - frontend request/response types use the same field names
  - update uses `PATCH /api/chores/{id}` consistently in spec, plan, and issue contract
