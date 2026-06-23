<script>
const API_BASE_URL = String(import.meta.env.VITE_YAMYAM_API_BASE_URL || "http://localhost:8080").replace(/\/$/, "");

function readStoredUser() {
  try {
    return JSON.parse(localStorage.getItem("yamyam.user") || "null");
  } catch (e) {
    localStorage.removeItem("yamyam.user");
    localStorage.removeItem("yamyam.token");
    return null;
  }
}

function clearStoredAuth() {
  localStorage.removeItem("yamyam.user");
  localStorage.removeItem("yamyam.token");
}

const api = {
  async request(url, options = {}) {
    const requestUrl = /^https?:\/\//i.test(url) ? url : `${API_BASE_URL}${url}`;
    const token = localStorage.getItem("yamyam.token") || "";
    const authHeaders = options.auth === false || !token ? {} : { Authorization: `Bearer ${token}` };
    const { auth, ...fetchOptions } = options;
    const response = await fetch(requestUrl, {
      headers: { "Content-Type": "application/json", ...authHeaders, ...(fetchOptions.headers || {}) },
      ...fetchOptions
    });

    if (!response.ok) {
      let message = "요청을 처리하지 못했습니다.";
      try {
        const error = await response.json();
        message = error.message || error.error || error.detail || message;
      } catch (e) {
        message = response.statusText || message;
      }
      throw new Error(message);
    }

    if (response.status === 204) return null;
    return response.json();
  },
  get(url) {
    return this.request(url);
  },
  post(url, body) {
    return this.request(url, { method: "POST", body: JSON.stringify(body || {}) });
  },
  put(url, body) {
    return this.request(url, { method: "PUT", body: JSON.stringify(body || {}) });
  },
  delete(url) {
    return this.request(url, { method: "DELETE" });
  }
};

const routes = {
  "/": "home",
  "/home": "home",
  "/auth/login": "login",
  "/auth/signup": "signup",
  "/meals": "meals",
  "/meals/new": "mealForm",
  "/meals/edit": "mealForm",
  "/meals/detail": "mealDetail",
  "/profile": "profile",
  "/social": "social",
  "/challenge": "challenge",
  "/community": "community",
  "/coach": "coach",
  "/coach-v2": "coachV2"
};

export default {
  data() {
    const initialRoute = routes[location.pathname] || "home";
    const storedUser = readStoredUser();
    const storedToken = localStorage.getItem("yamyam.token") || "";
    const isAuthRoute = ["login", "signup"].includes(initialRoute);
    return {
      route: storedUser && storedToken || isAuthRoute ? initialRoute : "login",
      query: new URLSearchParams(location.search),
      user: storedUser,
      token: storedToken,
      loading: false,
      toast: "",
      meals: [],
      foods: [],
      challenges: [],
      memberships: {},
      participants: {},
      social: { following: [], followers: [], suggestions: [], leaderboard: [] },
      community: { posts: [], comments: {}, authors: {}, category: "all" },
      analysis: null,
      advice: null,
      adviceV2: null,
      mealFilters: { startDate: "", endDate: "", mealType: "", sortKey: "dateDesc" },
      loginForm: { email: "demo@yamyam.com", password: "Demo1234!" },
      signupForm: {
        email: "",
        password: "",
        nickname: "",
        gender: "female",
        birthYear: 1998,
        height: 165,
        weight: 58,
        goal: "maintain",
        healthNote: ""
      },
      profileForm: {},
      mealForm: {
        id: "",
        mealDate: new Date().toISOString().slice(0, 10),
        mealType: "breakfast",
        memo: "",
        foods: []
      },
      foodKeyword: "",
      challengeForm: { title: "", description: "", category: "식단", targetCount: 7, endDate: "" },
      postForm: { category: "meal", linkedMealId: "", title: "", content: "" },
      commentDrafts: {},
      coachForm: { mealId: "", question: "최근 식단에서 개선할 점을 알려줘." },
      coachV2Form: {
        date: new Date().toISOString().slice(0, 10),
        city: "Seoul",
        question: "오늘 날씨와 내 식단을 바탕으로 저녁 식사와 운동을 추천해줘.",
        mode: "multi"
      }
    };
  },
  computed: {
    authenticated() {
      return !!this.user && !!this.token;
    },
    displayName() {
      return this.user && this.user.nickname ? this.user.nickname : "사용자";
    },
    pageTitle() {
      return {
        home: "대시보드",
        meals: "식단 기록",
        mealForm: this.mealForm.id ? "식단 수정" : "식단 등록",
        mealDetail: "식단 상세",
        profile: "프로필",
        social: "소셜",
        challenge: "챌린지",
        community: "커뮤니티",
        coach: "AI 코치",
        coachV2: "AI 코치 V2"
      }[this.route] || "YamYam Coach";
    },
    todaySummary() {
      const today = new Date().toISOString().slice(0, 10);
      return this.summarize(this.meals
        .filter(meal => meal.mealDate === today)
        .flatMap(meal => meal.foods || []));
    },
    dailyGoal() {
      if (!this.user) return { calories: 2000 };
      const height = Number(this.user.height) || 0;
      const weight = Number(this.user.weight) || 0;
      if (height <= 0 || weight <= 0) return { calories: 2000 };
      const age = Math.max(15, new Date().getFullYear() - Number(this.user.birthYear || 0));
      const gender = String(this.user.gender || "").toLowerCase();
      const isMale = gender === "m" || gender === "male" || this.user.gender === "남성";
      const bmr = 10 * weight + 6.25 * height - 5 * age + (isMale ? 5 : -161);
      const calories = Math.round(bmr * 1.4);
      return { calories: Math.max(1200, calories) };
    },
    caloriePct() {
      return this.dailyGoal.calories
        ? Math.min(100, Math.round(this.todaySummary.calories * 100 / this.dailyGoal.calories))
        : 0;
    },
    filteredMeals() {
      let result = [...this.meals];
      if (this.mealFilters.startDate) {
        result = result.filter(meal => meal.mealDate >= this.mealFilters.startDate);
      }
      if (this.mealFilters.endDate) {
        result = result.filter(meal => meal.mealDate <= this.mealFilters.endDate);
      }
      if (this.mealFilters.mealType) {
        result = result.filter(meal => meal.mealType === this.mealFilters.mealType);
      }
      const sortKey = this.mealFilters.sortKey;
      result.sort((a, b) => {
        if (sortKey === "dateAsc") return String(a.mealDate).localeCompare(String(b.mealDate));
        if (sortKey === "energyDesc") return this.mealCalories(b) - this.mealCalories(a);
        return String(b.mealDate).localeCompare(String(a.mealDate));
      });
      return result;
    },
    analysisRecommendations() {
      return this.analysis && this.analysis.recommendations ? this.analysis.recommendations : [];
    },
    analysisWarnings() {
      return this.analysis && this.analysis.warnings ? this.analysis.warnings : [];
    },
    adviceV2Plan() {
      return this.adviceV2 && this.adviceV2.executedPlan ? this.adviceV2.executedPlan : [];
    },
    adviceV2ToolResults() {
      return this.adviceV2 && this.adviceV2.toolResults ? this.adviceV2.toolResults : [];
    },
    selectedMeal() {
      if (!this.analysis) return null;
      return this.meals.find(meal => meal.id === this.analysis.mealId) || null;
    }
  },
  async mounted() {
    addEventListener("popstate", this.syncRoute);
    await this.loadRoute();
  },
  methods: {
    syncRoute() {
      this.route = routes[location.pathname] || "home";
      this.query = new URLSearchParams(location.search);
      this.loadRoute();
    },
    go(path) {
      history.pushState(null, "", path);
      this.syncRoute();
    },
    navClass(path) {
      return location.pathname === path ? "nav-link active" : "nav-link";
    },
    notify(message) {
      this.toast = message;
      setTimeout(() => {
        if (this.toast === message) this.toast = "";
      }, 3200);
    },
    requireAuth() {
      if (!this.authenticated && !["login", "signup"].includes(this.route)) {
        clearStoredAuth();
        this.user = null;
        this.token = "";
        this.go("/auth/login");
        return false;
      }
      return true;
    },
    async loadRoute() {
      if (!this.requireAuth()) return;
      this.loading = true;
      try {
        if (["home", "meals", "mealDetail", "mealForm", "profile", "community", "coach", "coachV2"].includes(this.route)) {
          await this.loadMeals();
        }
        if (this.route === "mealForm") await this.prepareMealForm();
        if (this.route === "mealDetail") await this.loadAnalysis();
        if (this.route === "profile") this.profileForm = { ...this.user };
        if (this.route === "social") await this.loadSocial();
        if (["challenge", "home", "coach"].includes(this.route)) await this.loadChallenges();
        if (this.route === "community") await this.loadCommunity();
      } catch (error) {
        this.notify(error.message);
      } finally {
        this.loading = false;
      }
    },
    async login() {
      try {
        const result = await api.request("/api/auth/login", {
          method: "POST",
          auth: false,
          body: JSON.stringify(this.loginForm)
        });
        this.user = result.user;
        this.token = result.accessToken || "";
        localStorage.setItem("yamyam.user", JSON.stringify(this.user));
        localStorage.setItem("yamyam.token", this.token);
        this.go("/home");
      } catch (error) {
        this.notify(error.message);
      }
    },
    async signup() {
      try {
        await api.request("/api/users", {
          method: "POST",
          auth: false,
          body: JSON.stringify(this.signupForm)
        });
        const result = await api.request("/api/auth/login", {
          method: "POST",
          auth: false,
          body: JSON.stringify({
            email: this.signupForm.email,
            password: this.signupForm.password
          })
        });
        this.user = result.user;
        this.token = result.accessToken || "";
        localStorage.setItem("yamyam.user", JSON.stringify(this.user));
        localStorage.setItem("yamyam.token", this.token);
        this.go("/home");
      } catch (error) {
        this.notify(error.message);
      }
    },
    async logout() {
      try {
        await api.post("/api/auth/logout", {});
      } catch (e) {
        // Client-side cleanup still matters even if logout endpoint is unavailable.
      }
      clearStoredAuth();
      this.user = null;
      this.token = "";
      this.go("/auth/login");
    },
    async loadMeals() {
      if (!this.user) return;
      this.meals = await api.get(`/api/meals?userId=${encodeURIComponent(this.user.id)}`);
    },
    async loadFoods() {
      const keyword = encodeURIComponent(this.foodKeyword || "");
      this.foods = await api.get(`/api/foods?keyword=${keyword}&limit=30`);
    },
    async prepareMealForm() {
      const mealId = this.query.get("mealId");
      if (mealId) {
        const meal = await api.get(`/api/meals/${mealId}`);
        this.mealForm = {
          id: meal.id,
          mealDate: meal.mealDate,
          mealType: meal.mealType,
          memo: meal.memo || "",
          foods: (meal.foods || []).map(food => ({ ...food, foodCode: food.code, grams: food.grams || 100 }))
        };
      } else {
        this.mealForm = {
          id: "",
          mealDate: new Date().toISOString().slice(0, 10),
          mealType: "breakfast",
          memo: "",
          foods: []
        };
      }
      await this.loadFoods();
    },
    addFood(food) {
      if (this.mealForm.foods.some(item => item.foodCode === food.code)) return;
      this.mealForm.foods.push({ ...food, foodCode: food.code, grams: food.grams || 100 });
    },
    removeFood(code) {
      this.mealForm.foods = this.mealForm.foods.filter(food => food.foodCode !== code);
    },
    async saveMeal() {
      const body = {
        mealDate: this.mealForm.mealDate,
        mealType: this.mealForm.mealType,
        memo: this.mealForm.memo,
        foods: this.mealForm.foods.map(food => ({
          foodCode: food.foodCode || food.code,
          grams: Number(food.grams) || 100
        }))
      };
      try {
        const saved = this.mealForm.id
          ? await api.put(`/api/meals/${this.mealForm.id}`, body)
          : await api.post("/api/meals", { ...body, userId: this.user.id });
        this.go(`/meals/detail?mealId=${saved.id}`);
      } catch (error) {
        this.notify(error.message);
      }
    },
    async deleteMeal(id) {
      if (!confirm("식단을 삭제할까요?")) return;
      await api.delete(`/api/meals/${id}`);
      this.notify("식단을 삭제했습니다.");
      this.go("/meals");
    },
    async loadAnalysis() {
      const mealId = this.query.get("mealId");
      this.analysis = mealId ? await api.get(`/api/meals/${mealId}/analysis`) : null;
    },
    async saveProfile() {
      try {
        const body = {
          nickname: this.profileForm.nickname,
          gender: this.profileForm.gender,
          birthYear: Number(this.profileForm.birthYear),
          height: Number(this.profileForm.height),
          weight: Number(this.profileForm.weight),
          goal: this.profileForm.goal,
          healthNote: this.profileForm.healthNote || ""
        };
        this.user = await api.put(`/api/users/${this.user.id}`, body);
        localStorage.setItem("yamyam.user", JSON.stringify(this.user));
        this.notify("프로필을 저장했습니다.");
      } catch (error) {
        this.notify(error.message);
      }
    },
    async deactivate() {
      if (!confirm("회원 정보를 비활성화할까요?")) return;
      await api.delete(`/api/users/${this.user.id}`);
      await this.logout();
    },
    async loadSocial() {
      this.social = await api.get(`/api/social?userId=${encodeURIComponent(this.user.id)}`);
    },
    async follow(id) {
      this.social = await api.post("/api/social/follow", { userId: this.user.id, targetUserId: id });
    },
    async unfollow(id) {
      this.social = await api.post("/api/social/unfollow", { userId: this.user.id, targetUserId: id });
    },
    async loadChallenges() {
      const data = await api.get(`/api/challenges?userId=${encodeURIComponent(this.user.id)}`);
      this.challenges = data.challenges || [];
      this.memberships = data.memberships || {};
      this.participants = data.participants || {};
    },
    async createChallenge() {
      const data = await api.post("/api/challenges", {
        ...this.challengeForm,
        userId: this.user.id,
        targetCount: Number(this.challengeForm.targetCount)
      });
      this.challenges = data.challenges || [];
      this.memberships = data.memberships || {};
      this.participants = data.participants || {};
      this.challengeForm = { title: "", description: "", category: "식단", targetCount: 7, endDate: "" };
    },
    async joinChallenge(id) {
      const data = await api.post(`/api/challenges/${id}/join`, { userId: this.user.id });
      this.challenges = data.challenges || [];
      this.memberships = data.memberships || {};
      this.participants = data.participants || {};
    },
    async updateChallengeProgress(id, progress) {
      const data = await api.post(`/api/challenges/${id}/progress`, { userId: this.user.id, progress: Number(progress) });
      this.challenges = data.challenges || [];
      this.memberships = data.memberships || {};
      this.participants = data.participants || {};
    },
    async loadCommunity() {
      const category = this.community.category || "all";
      const data = await api.get(`/api/community?category=${encodeURIComponent(category)}`);
      this.community = { ...data, category };
    },
    async createPost() {
      const data = await api.post("/api/community/posts", { ...this.postForm, userId: this.user.id });
      this.community = { ...data, category: "all" };
      this.postForm = { category: "meal", linkedMealId: "", title: "", content: "" };
    },
    async createComment(postId) {
      const content = (this.commentDrafts[postId] || "").trim();
      if (!content) return;
      const data = await api.post(`/api/community/posts/${postId}/comments`, { userId: this.user.id, content });
      this.community = { ...data, category: "all" };
      this.commentDrafts[postId] = "";
    },
    async askCoach() {
      try {
        this.advice = await api.post("/api/coach/advice", { userId: this.user.id, ...this.coachForm });
      } catch (error) {
        this.notify(error.message);
      }
    },
    async askCoachV2() {
      try {
        const endpoint = this.coachV2Form.mode === "llm-driven" ? "/api/v2/coach/llm-driven" : "/api/v2/coach/multi";
        this.adviceV2 = await api.post(endpoint, {
          userId: this.user.id,
          date: this.coachV2Form.date,
          city: this.coachV2Form.city,
          question: this.coachV2Form.question
        });
      } catch (error) {
        this.notify(error.message);
      }
    },
    mealLabel(type) {
      return { breakfast: "아침", lunch: "점심", dinner: "저녁", snack: "간식" }[type] || type;
    },
    goalLabel(goal) {
      return { loss: "감량", maintain: "유지", gain: "증량" }[goal] || goal;
    },
    categoryLabel(category) {
      return { meal: "식단", question: "질문", tip: "팁" }[category] || category;
    },
    mealCalories(meal) {
      return Math.round(Number(meal && meal.nutrition ? meal.nutrition.calories : 0));
    },
    mealFoodNames(meal) {
      const foods = meal && meal.foods ? meal.foods : [];
      if (!foods.length) return "음식 정보 없음";
      return foods.map(food => food.name).join(", ");
    },
    foodAmount(food) {
      return `${food.name} ${Math.round(Number(food.grams || 0))}g`;
    },
    authorName(userId) {
      const author = this.community.authors ? this.community.authors[userId] : null;
      return author && author.nickname ? author.nickname : userId;
    },
    summarize(foods) {
      const total = foods.reduce((acc, food) => {
        acc.calories += Number(food.energy || 0);
        acc.carbs += Number(food.carbs || 0);
        acc.protein += Number(food.protein || 0);
        acc.fat += Number(food.fat || 0);
        return acc;
      }, { calories: 0, carbs: 0, protein: 0, fat: 0 });
      const macroCalories = total.carbs * 4 + total.protein * 4 + total.fat * 9;
      total.carbsPct = macroCalories ? Math.round(total.carbs * 4 * 100 / macroCalories) : 0;
      total.proteinPct = macroCalories ? Math.round(total.protein * 4 * 100 / macroCalories) : 0;
      total.fatPct = macroCalories ? Math.max(0, 100 - total.carbsPct - total.proteinPct) : 0;
      total.calories = Math.round(total.calories);
      return total;
    },
    fmt(value) {
      return value ? String(value).replace("T", " ").slice(0, 16) : "";
    }
  }
};
</script>

<template>

<div v-if="route === 'login' || route === 'signup'" class="auth-page">
  <div class="auth-grid">
    <section class="auth-copy">
      <div class="eyebrow">YamYam Coach</div>
      <h1>식단 기록과 코칭을 한 화면에서 관리합니다</h1>
      <p class="muted mt-3 mb-0">REST API와 Vue.js로 구성된 냠냠코치 클라이언트입니다.</p>
    </section>

    <section class="auth-form" v-if="route === 'login'">
      <h2>로그인</h2>
      <form @submit.prevent="login" class="d-grid gap-3">
        <input class="form-control" v-model="loginForm.email" type="email" placeholder="이메일" required>
        <input class="form-control" v-model="loginForm.password" type="password" placeholder="비밀번호" required>
        <button class="btn btn-success">로그인</button>
        <button type="button" class="btn btn-outline-secondary" @click="go('/auth/signup')">회원가입</button>
      </form>
    </section>

    <section class="auth-form" v-else>
      <h2>회원가입</h2>
      <form @submit.prevent="signup" class="d-grid gap-3">
        <input class="form-control" v-model="signupForm.email" type="email" placeholder="이메일" required>
        <input class="form-control" v-model="signupForm.password" type="password" placeholder="비밀번호" required>
        <input class="form-control" v-model="signupForm.nickname" placeholder="닉네임" required>
        <div class="row g-2">
          <div class="col">
            <select class="form-select" v-model="signupForm.gender">
              <option value="female">여성</option>
              <option value="male">남성</option>
            </select>
          </div>
          <div class="col">
            <input class="form-control" v-model.number="signupForm.birthYear" type="number" placeholder="출생연도">
          </div>
        </div>
        <div class="row g-2">
          <div class="col"><input class="form-control" v-model.number="signupForm.height" type="number" step="0.1" placeholder="키"></div>
          <div class="col"><input class="form-control" v-model.number="signupForm.weight" type="number" step="0.1" placeholder="몸무게"></div>
        </div>
        <select class="form-select" v-model="signupForm.goal">
          <option value="loss">감량</option>
          <option value="maintain">유지</option>
          <option value="gain">증량</option>
        </select>
        <textarea class="form-control" v-model="signupForm.healthNote" rows="2" placeholder="건강 메모"></textarea>
        <button class="btn btn-success">가입하고 시작</button>
        <button type="button" class="btn btn-outline-secondary" @click="go('/auth/login')">로그인으로 이동</button>
      </form>
    </section>
  </div>
  <div v-if="toast" class="toast-line">{{ toast }}</div>
</div>

<div v-else>
  <nav class="navbar navbar-expand-lg app-nav navbar-dark">
    <div class="container">
      <a class="navbar-brand" href="/home" @click.prevent="go('/home')">냠냠코치</a>
      <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#mainNav">
        <span class="navbar-toggler-icon"></span>
      </button>
      <div class="collapse navbar-collapse" id="mainNav">
        <div class="navbar-nav me-auto ms-lg-3">
          <a :class="navClass('/home')" href="/home" @click.prevent="go('/home')">홈</a>
          <a :class="navClass('/meals')" href="/meals" @click.prevent="go('/meals')">식단</a>
          <a :class="navClass('/profile')" href="/profile" @click.prevent="go('/profile')">프로필</a>
          <a :class="navClass('/social')" href="/social" @click.prevent="go('/social')">소셜</a>
          <a :class="navClass('/challenge')" href="/challenge" @click.prevent="go('/challenge')">챌린지</a>
          <a :class="navClass('/community')" href="/community" @click.prevent="go('/community')">커뮤니티</a>
          <a :class="navClass('/coach')" href="/coach" @click.prevent="go('/coach')">코치</a>
          <a :class="navClass('/coach-v2')" href="/coach-v2" @click.prevent="go('/coach-v2')">코치 V2</a>
        </div>
        <div class="navbar-text me-3 d-none d-lg-block">{{ displayName }}</div>
        <button class="btn btn-outline-light btn-sm" @click="logout">
          <i class="bi bi-box-arrow-right me-1"></i>로그아웃
        </button>
      </div>
    </div>
  </nav>

  <main class="container app-shell">
    <div class="loading-line" v-if="loading">불러오는 중...</div>

    <section class="hero-panel mb-4" v-if="route === 'home'">
      <div class="panel-body page-title mb-0">
        <div>
          <div class="eyebrow text-white">Dashboard</div>
          <h1>{{ displayName }}님의 식단 대시보드</h1>
          <p class="muted mt-2 mb-0">오늘의 목표와 최근 기록을 확인합니다.</p>
        </div>
        <button class="btn btn-light" @click="go('/meals/new')">
          <i class="bi bi-plus-lg me-1"></i>식단 등록
        </button>
      </div>
    </section>

    <div class="page-title" v-else>
      <div>
        <div class="eyebrow">YamYam</div>
        <h1>{{ pageTitle }}</h1>
      </div>
      <button v-if="route === 'meals'" class="btn btn-success" @click="go('/meals/new')">
        <i class="bi bi-plus-lg me-1"></i>새 식단
      </button>
    </div>

    <section v-if="route === 'home'">
      <div class="metric-grid mb-4">
        <div class="metric">
          <div class="metric-label">오늘 칼로리</div>
          <div class="metric-value">{{ todaySummary.calories }}</div>
          <div class="muted">목표 {{ dailyGoal.calories }} kcal</div>
        </div>
        <div class="metric">
          <div class="metric-label">탄단지 비율</div>
          <div class="metric-value">{{ todaySummary.carbsPct }}/{{ todaySummary.proteinPct }}/{{ todaySummary.fatPct }}</div>
          <div class="muted">탄수화물 / 단백질 / 지방</div>
        </div>
        <div class="metric">
          <div class="metric-label">식단 기록</div>
          <div class="metric-value">{{ meals.length }}</div>
          <div class="muted">전체 기록</div>
        </div>
        <div class="metric">
          <div class="metric-label">참여 챌린지</div>
          <div class="metric-value">{{ Object.keys(memberships).length }}</div>
          <div class="muted">진행 중</div>
        </div>
      </div>

      <div class="two-grid mb-4">
        <div class="chart-panel">
          <div class="section-head"><h2>오늘 칼로리 진행률</h2></div>
          <div class="d-flex align-items-center gap-4">
            <div class="donut" :style="{ '--pct': caloriePct }"><div class="donut-inner">{{ caloriePct }}%</div></div>
            <div>
              <div class="metric-value">{{ todaySummary.calories }} kcal</div>
              <div class="muted">일일 목표 {{ dailyGoal.calories }} kcal</div>
            </div>
          </div>
        </div>
        <div class="chart-panel">
          <div class="section-head"><h2>탄단지 비율</h2></div>
          <div class="macro-bar mb-3">
            <div class="macro-carb" :style="{ width: todaySummary.carbsPct + '%' }"></div>
            <div class="macro-protein" :style="{ width: todaySummary.proteinPct + '%' }"></div>
            <div class="macro-fat" :style="{ width: todaySummary.fatPct + '%' }"></div>
          </div>
          <span class="badge-soft badge-accent me-2">탄수화물 {{ todaySummary.carbsPct }}%</span>
          <span class="badge-soft me-2">단백질 {{ todaySummary.proteinPct }}%</span>
          <span class="badge-soft badge-danger-soft">지방 {{ todaySummary.fatPct }}%</span>
        </div>
      </div>

      <div class="layout-grid">
        <div class="panel">
          <div class="panel-body">
            <div class="section-head">
              <h2>최근 식단</h2>
              <button class="btn btn-outline-success btn-sm" @click="go('/meals')">전체 보기</button>
            </div>
            <div v-if="!meals.length" class="empty-state">등록된 식단이 없습니다.</div>
            <a v-for="meal in filteredMeals.slice(0, 5)" :key="meal.id" class="meal-row" href="#" @click.prevent="go('/meals/detail?mealId=' + meal.id)">
              <span class="badge-soft">{{ mealLabel(meal.mealType) }}</span>
              <div>
                <strong>{{ meal.mealDate }}</strong>
                <div class="meal-foods">{{ mealFoodNames(meal) }}</div>
                <div class="muted">{{ (meal.foods || []).length }}개 음식 · {{ mealCalories(meal) }} kcal</div>
              </div>
              <i class="bi bi-chevron-right text-secondary"></i>
            </a>
          </div>
        </div>
        <aside class="panel">
          <div class="panel-body">
            <div class="section-head"><h2>활성 챌린지</h2></div>
            <div v-if="!challenges.length" class="empty-state">진행 중인 챌린지가 없습니다.</div>
            <a v-for="challenge in challenges.slice(0, 3)" :key="challenge.id" class="insight-box d-block mb-2" href="#" @click.prevent="go('/challenge')">
              <strong>{{ challenge.title }}</strong>
              <div class="muted mt-1">{{ challenge.description }}</div>
              <div class="muted mt-2">목표 {{ challenge.targetCount }}회 · 종료 {{ challenge.endDate }}</div>
            </a>
          </div>
        </aside>
      </div>
    </section>

    <section v-if="route === 'meals'">
      <div class="panel mb-4">
        <div class="panel-body">
          <div class="row g-3 align-items-end">
            <div class="col-md-3">
              <label class="form-label">시작일</label>
              <input class="form-control" type="date" v-model="mealFilters.startDate">
            </div>
            <div class="col-md-3">
              <label class="form-label">종료일</label>
              <input class="form-control" type="date" v-model="mealFilters.endDate">
            </div>
            <div class="col-md-3">
              <label class="form-label">식사 유형</label>
              <select class="form-select" v-model="mealFilters.mealType">
                <option value="">전체</option>
                <option value="breakfast">아침</option>
                <option value="lunch">점심</option>
                <option value="dinner">저녁</option>
                <option value="snack">간식</option>
              </select>
            </div>
            <div class="col-md-3">
              <label class="form-label">정렬</label>
              <select class="form-select" v-model="mealFilters.sortKey">
                <option value="dateDesc">최신순</option>
                <option value="dateAsc">오래된순</option>
                <option value="energyDesc">칼로리 높은순</option>
              </select>
            </div>
          </div>
        </div>
      </div>

      <div class="panel">
        <div class="panel-body">
          <div v-if="!filteredMeals.length" class="empty-state">조회된 식단이 없습니다.</div>
          <a v-for="meal in filteredMeals" :key="meal.id" class="meal-row" href="#" @click.prevent="go('/meals/detail?mealId=' + meal.id)">
            <span class="badge-soft">{{ mealLabel(meal.mealType) }}</span>
            <div>
              <strong>{{ meal.mealDate }}</strong>
              <div class="meal-foods">{{ mealFoodNames(meal) }}</div>
              <div class="muted">{{ (meal.foods || []).length }}개 음식 · {{ mealCalories(meal) }} kcal · {{ meal.memo }}</div>
            </div>
            <div class="row-actions">
              <button class="btn btn-outline-success btn-sm" @click.stop.prevent="go('/meals/edit?mealId=' + meal.id)">수정</button>
              <button class="btn btn-outline-danger btn-sm" @click.stop.prevent="deleteMeal(meal.id)">삭제</button>
            </div>
          </a>
        </div>
      </div>
    </section>

    <section v-if="route === 'mealForm'" class="two-grid">
      <form class="panel" @submit.prevent="saveMeal">
        <div class="panel-body d-grid gap-3">
          <input class="form-control" v-model="mealForm.mealDate" type="date" required>
          <select class="form-select" v-model="mealForm.mealType">
            <option value="breakfast">아침</option>
            <option value="lunch">점심</option>
            <option value="dinner">저녁</option>
            <option value="snack">간식</option>
          </select>
          <textarea class="form-control" v-model="mealForm.memo" rows="3" placeholder="메모"></textarea>
          <div>
            <h2 class="section-subtitle">선택한 음식</h2>
            <div v-if="!mealForm.foods.length" class="empty-state">음식을 선택하세요.</div>
            <div v-for="food in mealForm.foods" :key="food.foodCode" class="list-row">
              <span class="badge-soft">{{ food.category }}</span>
              <div>
                <strong>{{ food.name }}</strong>
                <input class="form-control mt-2" v-model.number="food.grams" type="number" step="1">
              </div>
              <button type="button" class="btn btn-outline-danger btn-sm" @click="removeFood(food.foodCode)">
                <i class="bi bi-x-lg"></i>
              </button>
            </div>
          </div>
          <button class="btn btn-success">저장</button>
        </div>
      </form>

      <div class="panel">
        <div class="panel-body">
          <div class="section-head"><h2>음식 검색</h2></div>
          <div class="input-group mb-3">
            <input class="form-control" v-model="foodKeyword" placeholder="음식명 또는 분류">
            <button class="btn btn-outline-success" @click="loadFoods">검색</button>
          </div>
          <div class="table-wrap">
            <table class="table align-middle">
              <thead><tr><th>음식</th><th>kcal</th><th></th></tr></thead>
              <tbody>
                <tr v-for="food in foods" :key="food.code">
                  <td><strong>{{ food.name }}</strong><div class="muted">{{ food.category }}</div></td>
                  <td>{{ Math.round(food.energy) }}</td>
                  <td><button class="btn btn-outline-success btn-sm" @click="addFood(food)">추가</button></td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </section>

    <section v-if="route === 'mealDetail'" class="layout-grid">
      <div class="panel" v-if="analysis">
        <div class="panel-body">
          <div class="section-head">
            <h2>분석 결과</h2>
            <div class="row-actions">
              <button class="btn btn-outline-success btn-sm" @click="go('/meals/edit?mealId=' + analysis.mealId)">수정</button>
              <button class="btn btn-outline-danger btn-sm" @click="deleteMeal(analysis.mealId)">삭제</button>
            </div>
          </div>
          <p>{{ analysis.summary }}</p>
          <div class="food-chip-list mb-3" v-if="selectedMeal">
            <span class="food-chip" v-for="food in selectedMeal.foods" :key="food.code">{{ foodAmount(food) }}</span>
          </div>
          <div class="macro-bar mb-3">
            <div class="macro-carb" :style="{ width: analysis.nutrition.carbsPct + '%' }"></div>
            <div class="macro-protein" :style="{ width: analysis.nutrition.proteinPct + '%' }"></div>
            <div class="macro-fat" :style="{ width: analysis.nutrition.fatPct + '%' }"></div>
          </div>
          <div class="metric-grid">
            <div class="metric"><div class="metric-label">칼로리</div><div class="metric-value">{{ Math.round(analysis.nutrition.calories) }}</div></div>
            <div class="metric"><div class="metric-label">탄수화물</div><div class="metric-value">{{ Math.round(analysis.nutrition.carbs) }}g</div></div>
            <div class="metric"><div class="metric-label">단백질</div><div class="metric-value">{{ Math.round(analysis.nutrition.protein) }}g</div></div>
            <div class="metric"><div class="metric-label">지방</div><div class="metric-value">{{ Math.round(analysis.nutrition.fat) }}g</div></div>
          </div>
        </div>
      </div>
      <aside class="panel">
        <div class="panel-body">
          <h2 class="section-subtitle">권장 사항</h2>
          <div v-if="!analysis" class="empty-state">식단을 찾을 수 없습니다.</div>
          <div v-for="item in analysisRecommendations" class="insight-box mb-2">{{ item }}</div>
          <div v-for="item in analysisWarnings" class="insight-box warning-box mb-2">{{ item }}</div>
        </div>
      </aside>
    </section>

    <section v-if="route === 'profile'" class="panel">
      <form class="panel-body d-grid gap-3" @submit.prevent="saveProfile">
        <div class="row g-3">
          <div class="col-md-6"><label class="form-label">이메일</label><input class="form-control" v-model="profileForm.email" disabled></div>
          <div class="col-md-6"><label class="form-label">닉네임</label><input class="form-control" v-model="profileForm.nickname" required></div>
          <div class="col-md-4">
            <label class="form-label">성별</label>
            <select class="form-select" v-model="profileForm.gender"><option value="female">여성</option><option value="male">남성</option></select>
          </div>
          <div class="col-md-4"><label class="form-label">출생연도</label><input class="form-control" v-model.number="profileForm.birthYear" type="number"></div>
          <div class="col-md-4">
            <label class="form-label">목표</label>
            <select class="form-select" v-model="profileForm.goal"><option value="loss">감량</option><option value="maintain">유지</option><option value="gain">증량</option></select>
          </div>
          <div class="col-md-6"><label class="form-label">키</label><input class="form-control" v-model.number="profileForm.height" type="number" step="0.1"></div>
          <div class="col-md-6"><label class="form-label">몸무게</label><input class="form-control" v-model.number="profileForm.weight" type="number" step="0.1"></div>
        </div>
        <textarea class="form-control" v-model="profileForm.healthNote" rows="3" placeholder="건강 메모"></textarea>
        <div class="row-actions">
          <button class="btn btn-success">저장</button>
          <button type="button" class="btn btn-outline-danger" @click="deactivate">탈퇴</button>
        </div>
      </form>
    </section>

    <section v-if="route === 'social'" class="social-grid">
      <div class="panel">
        <div class="panel-body">
          <h2 class="section-subtitle">추천 사용자</h2>
          <div v-if="!social.suggestions.length" class="empty-state">추천 사용자가 없습니다.</div>
          <div v-for="target in social.suggestions" :key="target.id" class="list-row compact">
            <span class="avatar">{{ target.nickname.slice(0, 1) }}</span>
            <div><strong>{{ target.nickname }}</strong><div class="muted">{{ goalLabel(target.goal) }}</div></div>
            <button class="btn btn-outline-success btn-sm" @click="follow(target.id)">팔로우</button>
          </div>
        </div>
      </div>
      <div class="panel">
        <div class="panel-body">
          <h2 class="section-subtitle">팔로잉</h2>
          <div v-if="!social.following.length" class="empty-state">팔로잉이 없습니다.</div>
          <div v-for="target in social.following" :key="target.id" class="list-row compact">
            <span class="avatar">{{ target.nickname.slice(0, 1) }}</span>
            <div><strong>{{ target.nickname }}</strong><div class="muted">{{ goalLabel(target.goal) }}</div></div>
            <button class="btn btn-outline-secondary btn-sm" @click="unfollow(target.id)">해제</button>
          </div>
        </div>
      </div>
      <div class="panel">
        <div class="panel-body">
          <h2 class="section-subtitle">팔로워</h2>
          <div v-if="!social.followers.length" class="empty-state">팔로워가 없습니다.</div>
          <div v-for="target in social.followers" :key="target.id" class="list-row compact">
            <span class="avatar">{{ target.nickname.slice(0, 1) }}</span>
            <div><strong>{{ target.nickname }}</strong><div class="muted">{{ goalLabel(target.goal) }}</div></div>
          </div>
        </div>
      </div>
      <div class="panel">
        <div class="panel-body">
          <h2 class="section-subtitle">리더보드</h2>
          <div v-for="target in social.leaderboard" :key="target.id" class="list-row compact">
            <span class="avatar">{{ target.nickname.slice(0, 1) }}</span>
            <div><strong>{{ target.nickname }}</strong><div class="muted">{{ goalLabel(target.goal) }}</div></div>
          </div>
        </div>
      </div>
    </section>

    <section v-if="route === 'challenge'" class="layout-grid">
      <div class="panel">
        <div class="panel-body">
          <div v-if="!challenges.length" class="empty-state">등록된 챌린지가 없습니다.</div>
          <article v-for="challenge in challenges" :key="challenge.id" class="post-card">
            <div class="section-head">
              <div>
                <span class="badge-soft">{{ challenge.category }}</span>
                <h2>{{ challenge.title }}</h2>
                <p class="muted mb-0">{{ challenge.description }}</p>
              </div>
              <button v-if="!memberships[challenge.id]" class="btn btn-outline-success btn-sm" @click="joinChallenge(challenge.id)">참여</button>
            </div>
            <div v-if="memberships[challenge.id]">
              <div class="bar-track mb-2">
                <div class="bar-fill" :style="{ width: Math.min(100, memberships[challenge.id].progress * 100 / challenge.targetCount) + '%' }"></div>
              </div>
              <input class="form-range" type="range" min="0" :max="challenge.targetCount" :value="memberships[challenge.id].progress" @change="updateChallengeProgress(challenge.id, $event.target.value)">
              <div class="muted">{{ memberships[challenge.id].progress }} / {{ challenge.targetCount }}</div>
            </div>
          </article>
        </div>
      </div>
      <form class="panel" @submit.prevent="createChallenge">
        <div class="panel-body d-grid gap-3">
          <h2 class="section-subtitle">챌린지 생성</h2>
          <input class="form-control" v-model="challengeForm.title" placeholder="제목" required>
          <textarea class="form-control" v-model="challengeForm.description" rows="3" placeholder="설명"></textarea>
          <input class="form-control" v-model="challengeForm.category" placeholder="분류">
          <input class="form-control" v-model.number="challengeForm.targetCount" type="number" min="1">
          <input class="form-control" v-model="challengeForm.endDate" type="date" required>
          <button class="btn btn-success">생성</button>
        </div>
      </form>
    </section>

    <section v-if="route === 'community'" class="layout-grid">
      <div class="panel">
        <div class="panel-body">
          <div class="section-head">
            <h2>게시글</h2>
            <select class="form-select w-auto" v-model="community.category" @change="loadCommunity">
              <option value="all">전체</option>
              <option value="meal">식단</option>
              <option value="question">질문</option>
              <option value="tip">팁</option>
            </select>
          </div>
          <div v-if="!community.posts.length" class="empty-state">게시글이 없습니다.</div>
          <article v-for="post in community.posts" :key="post.id" class="post-card">
            <span class="badge-soft">{{ categoryLabel(post.category) }}</span>
            <h2>{{ post.title }}</h2>
            <p>{{ post.content }}</p>
            <div class="muted mb-3">{{ authorName(post.userId) }} · {{ fmt(post.createdAt) }}</div>
            <div class="insight-box mb-2" v-for="comment in community.comments[post.id] || []" :key="comment.id">
              <strong>{{ authorName(comment.userId) }}</strong>
              <div>{{ comment.content }}</div>
            </div>
            <form class="input-group" @submit.prevent="createComment(post.id)">
              <input class="form-control" v-model="commentDrafts[post.id]" placeholder="댓글">
              <button class="btn btn-outline-success">등록</button>
            </form>
          </article>
        </div>
      </div>
      <form class="panel" @submit.prevent="createPost">
        <div class="panel-body d-grid gap-3">
          <h2 class="section-subtitle">글 작성</h2>
          <select class="form-select" v-model="postForm.category">
            <option value="meal">식단</option>
            <option value="question">질문</option>
            <option value="tip">팁</option>
          </select>
          <select class="form-select" v-model="postForm.linkedMealId">
            <option value="">연결 식단 없음</option>
            <option v-for="meal in meals" :key="meal.id" :value="meal.id">{{ meal.mealDate }} {{ mealLabel(meal.mealType) }}</option>
          </select>
          <input class="form-control" v-model="postForm.title" placeholder="제목" required>
          <textarea class="form-control" v-model="postForm.content" rows="5" placeholder="내용" required></textarea>
          <button class="btn btn-success">게시</button>
        </div>
      </form>
    </section>

    <section v-if="route === 'coach'" class="two-grid">
      <form class="panel" @submit.prevent="askCoach">
        <div class="panel-body d-grid gap-3">
          <h2 class="section-subtitle">AI 코치 질문</h2>
          <select class="form-select" v-model="coachForm.mealId">
            <option value="">최근 식단 전체</option>
            <option v-for="meal in meals" :key="meal.id" :value="meal.id">{{ meal.mealDate }} {{ mealLabel(meal.mealType) }}</option>
          </select>
          <textarea class="form-control" v-model="coachForm.question" rows="5"></textarea>
          <button class="btn btn-success">조언 받기</button>
        </div>
      </form>
      <div class="panel">
        <div class="panel-body">
          <h2 class="section-subtitle">응답</h2>
          <div class="d-flex gap-2 mb-3" v-if="advice">
            <span class="badge-soft">{{ advice.provider }}</span>
            <span class="badge-soft badge-accent" v-if="advice.springAiUsed">Spring AI</span>
          </div>
          <div class="api-result" v-if="advice">{{ advice.advice }}</div>
          <div class="empty-state" v-else>질문을 보내면 코치 응답이 표시됩니다.</div>
        </div>
      </div>
    </section>

    <section v-if="route === 'coachV2'" class="two-grid">
      <form class="panel" @submit.prevent="askCoachV2">
        <div class="panel-body d-grid gap-3">
          <h2 class="section-subtitle">AI 코치 V2</h2>
          <div class="row g-2">
            <div class="col"><label class="form-label">날짜</label><input class="form-control" v-model="coachV2Form.date" type="date"></div>
            <div class="col"><label class="form-label">도시</label><input class="form-control" v-model="coachV2Form.city" placeholder="Seoul"></div>
          </div>
          <select class="form-select" v-model="coachV2Form.mode">
            <option value="multi">다중 Tool 연쇄</option>
            <option value="llm-driven">LLM 주도 Tool 선택</option>
          </select>
          <textarea class="form-control" v-model="coachV2Form.question" rows="5"></textarea>
          <button class="btn btn-success">코칭 요청</button>
        </div>
      </form>
      <div class="panel">
        <div class="panel-body">
          <h2 class="section-subtitle">V2 응답</h2>
          <template v-if="adviceV2">
            <div class="d-flex flex-wrap gap-2 mb-3">
              <span class="badge-soft badge-accent">{{ adviceV2.mode }}</span>
              <span class="badge-soft">{{ adviceV2.provider }}</span>
              <span class="badge-soft" v-if="adviceV2.springAiUsed">Spring AI</span>
            </div>
            <div class="api-result mb-3">{{ adviceV2.answer }}</div>
            <div v-if="adviceV2Plan.length">
              <h3 class="section-subtitle">실행 계획</h3>
              <ol class="muted ps-3"><li v-for="step in adviceV2Plan" :key="step">{{ step }}</li></ol>
            </div>
            <div v-if="adviceV2ToolResults.length">
              <h3 class="section-subtitle mt-3">Tool 실행 결과</h3>
              <div v-for="tool in adviceV2ToolResults" :key="tool.toolName + tool.summary" class="insight-box mb-2">
                <div class="d-flex justify-content-between">
                  <strong>{{ tool.toolName }}</strong>
                  <span :class="tool.success ? 'badge-soft badge-accent' : 'badge-soft badge-danger-soft'">{{ tool.success ? '성공' : '실패' }}</span>
                </div>
                <div class="muted mt-1">{{ tool.summary }}</div>
              </div>
            </div>
          </template>
          <div class="empty-state" v-else>코칭 요청을 보내면 결과가 표시됩니다.</div>
        </div>
      </div>
    </section>
  </main>
  <div v-if="toast" class="toast-line">{{ toast }}</div>
</div>
</template>
