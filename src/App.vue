<script>
const API_BASE_URL = String(import.meta.env.VITE_YAMYAM_API_BASE_URL || "http://localhost:8080").replace(/\/$/, "");

function readStoredUser() {
  try {
    localStorage.removeItem("yamyam.token");
    return JSON.parse(localStorage.getItem("yamyam.user") || "null");
  } catch (e) {
    localStorage.removeItem("yamyam.user");
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
    const { headers, ...fetchOptions } = options;
    const response = await fetch(requestUrl, {
      ...fetchOptions,
      credentials: "include",
      headers: { "Content-Type": "application/json", ...(headers || {}) }
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
  "/reports": "reports",
  "/meals/new": "mealForm",
  "/meals/edit": "mealForm",
  "/meals/detail": "mealDetail",
  "/profile": "profile",
  "/social": "social",
  "/challenge": "challenge",
  "/community": "community",
  "/coach": "coach",
  "/coach-v2": "coachV2",
};

export default {
  data() {
    const initialRoute = routes[location.pathname] || "home";
    const storedUser = readStoredUser();
    const isAuthRoute = ["login", "signup"].includes(initialRoute);
    return {
      route: storedUser || isAuthRoute ? initialRoute : "login",
      query: new URLSearchParams(location.search),
      user: storedUser,
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
      coachLoading: false,
      coachV2Loading: false,
      reportLoading: false,
      mealFilters: { startDate: "", endDate: "", mealType: "", sortKey: "dateDesc" },
      reportForm: { date: new Date().toISOString().slice(0, 10) },
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
      challengeCreateOpen: false,
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
      return !!this.user;
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
        reports: "Reports",
        social: "소셜",
        challenge: "챌린지",
        community: "커뮤니티",
        coach: "AI 코치",
        coachV2: "AI 코치 V2",
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
    calorieRemaining() {
      return Math.max(0, this.dailyGoal.calories - this.todaySummary.calories);
    },
    todayMeals() {
      const today = new Date().toISOString().slice(0, 10);
      return this.meals.filter(meal => meal.mealDate === today);
    },
    joinedChallenges() {
      return this.challenges.filter(challenge => this.memberships[challenge.id]);
    },
    availableChallenges() {
      return this.challenges.filter(challenge => !this.memberships[challenge.id]);
    },
    macroTip() {
      if (!this.todaySummary.calories) return "오늘 첫 식단을 기록하면 탄단지 균형을 바로 확인할 수 있어요.";
      if (this.todaySummary.proteinPct >= 30) return "단백질 섭취가 좋아요. 남은 식사는 채소와 건강한 지방을 더해보세요.";
      if (this.todaySummary.carbsPct >= 60) return "탄수화물 비율이 높은 편이에요. 다음 식사는 단백질을 조금 더 챙겨보세요.";
      return "오늘의 탄단지 균형이 안정적이에요. 다음 식단도 같은 흐름으로 이어가보세요.";
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
    filteredMealSummary() {
      const totals = this.filteredMeals.reduce((acc, meal) => {
        const nutrition = meal.nutrition || {};
        acc.calories += Number(nutrition.calories || 0);
        acc.carbs += Number(nutrition.carbs || 0);
        acc.protein += Number(nutrition.protein || 0);
        acc.fat += Number(nutrition.fat || 0);
        return acc;
      }, { calories: 0, carbs: 0, protein: 0, fat: 0 });
      const count = this.filteredMeals.length;
      return {
        count,
        calories: Math.round(totals.calories),
        avgCalories: count ? Math.round(totals.calories / count) : 0,
        carbs: Math.round(totals.carbs),
        protein: Math.round(totals.protein),
        fat: Math.round(totals.fat)
      };
    },
    mealFormSummary() {
      const totals = this.mealForm.foods.reduce((acc, food) => {
        const ratio = (Number(food.grams) || 100) / 100;
        acc.calories += Number(food.energy || 0) * ratio;
        acc.carbs += Number(food.carbs || 0) * ratio;
        acc.protein += Number(food.protein || 0) * ratio;
        acc.fat += Number(food.fat || 0) * ratio;
        return acc;
      }, { calories: 0, carbs: 0, protein: 0, fat: 0 });
      return {
        calories: Math.round(totals.calories),
        carbs: Math.round(totals.carbs),
        protein: Math.round(totals.protein),
        fat: Math.round(totals.fat)
      };
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
        this.go("/auth/login");
        return false;
      }
      return true;
    },
    async loadRoute() {
      if (!this.requireAuth()) return;
      this.loading = true;
      try {
        if (["home", "meals", "mealDetail", "mealForm", "profile", "community", "coach", "coachV2", "reports"].includes(this.route)) {
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
          body: JSON.stringify(this.loginForm)
        });
        this.user = result.user;
        localStorage.setItem("yamyam.user", JSON.stringify(this.user));
        localStorage.removeItem("yamyam.token");
        this.go("/home");
      } catch (error) {
        this.notify(error.message);
      }
    },
    async signup() {
      try {
        await api.request("/api/users", {
          method: "POST",
          body: JSON.stringify(this.signupForm)
        });
        const result = await api.request("/api/auth/login", {
          method: "POST",
          body: JSON.stringify({
            email: this.signupForm.email,
            password: this.signupForm.password
          })
        });
        this.user = result.user;
        localStorage.setItem("yamyam.user", JSON.stringify(this.user));
        localStorage.removeItem("yamyam.token");
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
      try {
        const data = await api.post("/api/challenges", {
          ...this.challengeForm,
          userId: this.user.id,
          targetCount: Number(this.challengeForm.targetCount)
        });
        this.applyChallengeData(data);
        this.challengeForm = { title: "", description: "", category: "식단", targetCount: 7, endDate: "" };
        this.notify("챌린지를 생성했습니다.");
      } catch (error) {
        this.notify(error.message);
      }
    },
    async joinChallenge(id) {
      try {
        const data = await api.post(`/api/challenges/${id}/join`, { userId: this.user.id });
        this.applyChallengeData(data);
        this.notify("챌린지에 참여했습니다.");
      } catch (error) {
        this.notify(error.message);
      }
    },
    async updateChallengeProgress(id, progress) {
      try {
        const data = await api.post(`/api/challenges/${id}/progress`, { userId: this.user.id, progress: Number(progress) });
        this.applyChallengeData(data);
      } catch (error) {
        this.notify(error.message);
      }
    },
    applyChallengeData(data) {
      this.challenges = data.challenges || [];
      this.memberships = data.memberships || {};
      this.participants = data.participants || {};
    },
    challengeProgress(challenge) {
      const membership = this.memberships[challenge.id];
      return membership ? Number(membership.progress || 0) : 0;
    },
    challengePercent(challenge) {
      const target = Math.max(1, Number(challenge.targetCount || 1));
      return Math.min(100, Math.round(this.challengeProgress(challenge) * 100 / target));
    },
    challengeDday(challenge) {
      if (!challenge.endDate) return "기간 미정";
      const today = new Date();
      today.setHours(0, 0, 0, 0);
      const end = new Date(`${challenge.endDate}T00:00:00`);
      const diff = Math.ceil((end - today) / 86400000);
      if (diff < 0) return "종료";
      if (diff === 0) return "D-Day";
      return `D-${diff}`;
    },
    challengeParticipantCount(challenge) {
      return (this.participants[challenge.id] || []).length;
    },
    challengeIconClass(category) {
      const value = String(category || "");
      if (value.includes("영양") || value.includes("단백질")) return "bi-egg-fried tone-orange";
      if (value.includes("운동")) return "bi-heart-pulse tone-rose";
      if (value.includes("습관")) return "bi-moon-stars tone-purple";
      return "bi-basket2 tone-green";
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
      this.coachLoading = true;
      try {
        this.advice = await api.post("/api/coach/advice", { userId: this.user.id, ...this.coachForm });
      } catch (error) {
        this.notify(error.message);
      } finally {
        this.coachLoading = false;
      }
    },
    async askCoachV2() {
      this.coachV2Loading = true;
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
      } finally {
        this.coachV2Loading = false;
      }
    },
    reportMeals() {
      const date = this.reportForm.date;
      const rank = { breakfast: 1, lunch: 2, dinner: 3, snack: 4 };
      return this.meals
        .filter(meal => meal.mealDate === date)
        .sort((a, b) => (rank[a.mealType] || 9) - (rank[b.mealType] || 9));
    },
    reportSummary(meals) {
      return meals.reduce((acc, meal) => {
        const nutrition = meal.nutrition || {};
        acc.calories += Number(nutrition.calories || 0);
        acc.carbs += Number(nutrition.carbs || 0);
        acc.protein += Number(nutrition.protein || 0);
        acc.fat += Number(nutrition.fat || 0);
        return acc;
      }, { calories: 0, carbs: 0, protein: 0, fat: 0 });
    },
    async generateDailyReport() {
      const meals = this.reportMeals();
      if (!meals.length) {
        this.notify("선택한 날짜에 식단 기록이 없습니다.");
        return;
      }

      this.reportLoading = true;
      try {
        const analyses = [];
        for (const meal of meals) {
          const result = await api.post("/api/coach/advice", {
            userId: this.user.id,
            mealId: meal.id,
            question: `${this.reportForm.date} ${this.mealLabel(meal.mealType)} 식단을 일일 리포트용으로 간결하게 분석해줘.`
          });
          analyses.push({
            label: this.mealLabel(meal.mealType),
            advice: result.advice || ""
          });
        }

        await this.downloadReportPdf({
          date: this.reportForm.date,
          userName: this.user && (this.user.nickname || this.user.id) ? (this.user.nickname || this.user.id) : "user-demo",
          meals,
          summary: this.reportSummary(meals),
          analyses
        });
        this.notify("일일 식단 리포트를 다운로드했습니다.");
      } catch (error) {
        this.notify(error.message);
      } finally {
        this.reportLoading = false;
      }
    },
    async downloadReportPdf(report) {
      const canvas = document.createElement("canvas");
      const scale = 2;
      const width = 794;
      const height = 1123;
      canvas.width = width * scale;
      canvas.height = height * scale;
      const ctx = canvas.getContext("2d");
      ctx.scale(scale, scale);
      ctx.textBaseline = "top";

      const page = { x: 56, y: 54, w: width - 112 };
      let y = page.y;
      const colors = {
        ink: "#17201f",
        muted: "#61706d",
        brand: "#006a63",
        brandDark: "#004f4a",
        brandSoft: "#e7f5ef",
        amber: "#fff2d8",
        amberText: "#8a4d00",
        blue: "#e8f1ff",
        blueText: "#2557a7",
        rose: "#fdeceb",
        roseText: "#9b2f28",
        line: "#d7e4df",
        panel: "#ffffff",
        bg: "#f6faf8"
      };
      const font = (size, weight = "400") => `${weight} ${size}px "Malgun Gothic", "Apple SD Gothic Neo", Arial, sans-serif`;
      const n1 = value => Number(value || 0).toFixed(1);
      const formatDate = date => {
        const [yy, mm, dd] = String(date).split("-");
        return `${yy}년 ${mm}월 ${dd}일`;
      };
      const fillRound = (x, y, w, h, r, fill, stroke = "") => {
        ctx.beginPath();
        ctx.moveTo(x + r, y);
        ctx.arcTo(x + w, y, x + w, y + h, r);
        ctx.arcTo(x + w, y + h, x, y + h, r);
        ctx.arcTo(x, y + h, x, y, r);
        ctx.arcTo(x, y, x + w, y, r);
        ctx.closePath();
        ctx.fillStyle = fill;
        ctx.fill();
        if (stroke) {
          ctx.strokeStyle = stroke;
          ctx.lineWidth = 1;
          ctx.stroke();
        }
      };
      const text = (value, x, y, size = 16, weight = "400", color = colors.ink) => {
        ctx.font = font(size, weight);
        ctx.fillStyle = color;
        ctx.fillText(value, x, y);
      };
      const wrapLines = (value, maxWidth, size = 15, weight = "400") => {
        ctx.font = font(size, weight);
        const source = String(value || "").replace(/\s+/g, " ").trim();
        if (!source) return [];
        const words = source.split(" ");
        const lines = [];
        let current = "";
        for (const word of words) {
          const next = current ? `${current} ${word}` : word;
          if (ctx.measureText(next).width > maxWidth && current) {
            lines.push(current);
            current = word;
          } else {
            current = next;
          }
        }
        if (current) lines.push(current);
        return lines;
      };
      const sectionTitle = (label, icon, yPos) => {
        text(icon, page.x, yPos + 1, 18, "700", colors.brand);
        text(label, page.x + 28, yPos, 21, "800", colors.ink);
        ctx.strokeStyle = colors.line;
        ctx.lineWidth = 1;
        ctx.beginPath();
        ctx.moveTo(page.x, yPos + 34);
        ctx.lineTo(page.x + page.w, yPos + 34);
        ctx.stroke();
        return yPos + 54;
      };

      ctx.fillStyle = colors.bg;
      ctx.fillRect(0, 0, width, height);

      fillRound(page.x, y, page.w, 118, 18, colors.brand);
      text("YamYam Coach", page.x + 28, y + 24, 17, "700", "#d7fff4");
      text("일일 식단 리포트", page.x + 28, y + 51, 32, "800", "#ffffff");
      fillRound(page.x + page.w - 214, y + 28, 176, 54, 12, "rgba(255,255,255,0.16)");
      text(formatDate(report.date), page.x + page.w - 194, y + 42, 16, "800", "#ffffff");
      text(`사용자  ${report.userName}`, page.x + page.w - 194, y + 64, 13, "600", "#d7fff4");
      y += 148;

      y = sectionTitle("영양 섭취 현황", "01", y);
      const cards = [
        { label: "칼로리", value: `${n1(report.summary.calories)}`, unit: "kcal", bg: colors.brandSoft, fg: colors.brandDark },
        { label: "탄수화물", value: `${n1(report.summary.carbs)}`, unit: "g", bg: colors.amber, fg: colors.amberText },
        { label: "단백질", value: `${n1(report.summary.protein)}`, unit: "g", bg: colors.blue, fg: colors.blueText },
        { label: "지방", value: `${n1(report.summary.fat)}`, unit: "g", bg: colors.rose, fg: colors.roseText }
      ];
      const cardGap = 12;
      const cardW = (page.w - cardGap * 3) / 4;
      cards.forEach((card, index) => {
        const cx = page.x + index * (cardW + cardGap);
        fillRound(cx, y, cardW, 92, 14, card.bg, colors.line);
        text(card.label, cx + 18, y + 17, 14, "800", card.fg);
        text(card.value, cx + 18, y + 43, 24, "800", colors.ink);
        text(card.unit, cx + 18 + ctx.measureText(card.value).width + 8, y + 52, 13, "700", colors.muted);
      });
      y += 126;

      y = sectionTitle("기록된 식사", "02", y);
      const mealBoxH = Math.max(112, 30 + report.meals.length * 42);
      fillRound(page.x, y, page.w, mealBoxH, 14, colors.panel, colors.line);
      let mealY = y + 18;
      for (const meal of report.meals) {
        const foodNames = (meal.foods || []).map(food => food.name).join(", ") || "음식 정보 없음";
        const nutrition = meal.nutrition || {};
        fillRound(page.x + 18, mealY, 74, 28, 14, colors.brandSoft);
        text(this.mealLabel(meal.mealType), page.x + 38, mealY + 6, 13, "800", colors.brandDark);
        const lines = wrapLines(foodNames, 390, 14, "600");
        text(lines[0] || foodNames, page.x + 108, mealY + 2, 14, "700", colors.ink);
        if (lines[1]) text(lines[1], page.x + 108, mealY + 22, 13, "500", colors.muted);
        text(`${n1(nutrition.calories)} kcal`, page.x + page.w - 132, mealY + 2, 14, "800", colors.brand);
        text(`탄 ${n1(nutrition.carbs)}g · 단 ${n1(nutrition.protein)}g · 지 ${n1(nutrition.fat)}g`, page.x + page.w - 210, mealY + 23, 12, "600", colors.muted);
        mealY += 42;
      }
      y += mealBoxH + 28;

      y = sectionTitle("AI 코치 분석", "03", y);
      report.analyses.forEach((analysis, index) => {
        const allLines = wrapLines(analysis.advice, page.w - 48, 15, "400");
        const lines = allLines.length > 7 ? [...allLines.slice(0, 6), `${allLines[6]} ...`] : allLines;
        const cardH = Math.max(92, 58 + lines.length * 22);
        const tone = [colors.brandSoft, colors.blue, colors.amber, colors.rose][index % 4];
        fillRound(page.x, y, page.w, cardH, 14, colors.panel, colors.line);
        fillRound(page.x, y, 8, cardH, 4, tone);
        fillRound(page.x + 22, y + 18, 86, 30, 15, tone);
        text(analysis.label, page.x + 46, y + 25, 13, "800", colors.brandDark);
        let lineY = y + 56;
        lines.forEach(line => {
          text(line, page.x + 24, lineY, 15, "400", colors.ink);
          lineY += 22;
        });
        y += cardH + 14;
      });

      text("YamYam Coach daily report", page.x, height - 42, 11, "600", colors.muted);
      text("Generated from selected meal records and V1 coach analysis.", page.x + 438, height - 42, 11, "600", colors.muted);

      const jpegDataUrl = canvas.toDataURL("image/jpeg", 0.94);
      const pdfBytes = this.buildImagePdf(jpegDataUrl, 595.28, 841.89, canvas.width, canvas.height);
      const blob = new Blob([pdfBytes], { type: "application/pdf" });
      const url = URL.createObjectURL(blob);
      const link = document.createElement("a");
      link.href = url;
      link.download = `yamyam-daily-report-${report.date}.pdf`;
      document.body.appendChild(link);
      link.click();
      link.remove();
      URL.revokeObjectURL(url);
    },
    buildImagePdf(dataUrl, pageWidth, pageHeight, imageWidth, imageHeight) {
      const encoder = new TextEncoder();
      const imageBytes = Uint8Array.from(atob(dataUrl.split(",")[1]), char => char.charCodeAt(0));
      const parts = [];
      const offsets = [0];
      let length = 0;
      const push = value => {
        const bytes = typeof value === "string" ? encoder.encode(value) : value;
        parts.push(bytes);
        length += bytes.length;
      };
      const object = (id, body) => {
        offsets[id] = length;
        push(`${id} 0 obj\n`);
        push(body);
        push("\nendobj\n");
      };

      push("%PDF-1.4\n");
      object(1, "<< /Type /Catalog /Pages 2 0 R >>");
      object(2, "<< /Type /Pages /Kids [3 0 R] /Count 1 >>");
      object(3, `<< /Type /Page /Parent 2 0 R /MediaBox [0 0 ${pageWidth} ${pageHeight}] /Resources << /XObject << /Im1 4 0 R >> >> /Contents 5 0 R >>`);
      offsets[4] = length;
      push("4 0 obj\n");
      push(`<< /Type /XObject /Subtype /Image /Width ${imageWidth} /Height ${imageHeight} /ColorSpace /DeviceRGB /BitsPerComponent 8 /Filter /DCTDecode /Length ${imageBytes.length} >>\nstream\n`);
      push(imageBytes);
      push("\nendstream\nendobj\n");
      const content = `q\n${pageWidth} 0 0 ${pageHeight} 0 0 cm\n/Im1 Do\nQ`;
      object(5, `<< /Length ${content.length} >>\nstream\n${content}\nendstream`);

      const xrefOffset = length;
      push("xref\n0 6\n0000000000 65535 f \n");
      for (let i = 1; i <= 5; i += 1) {
        push(`${String(offsets[i]).padStart(10, "0")} 00000 n \n`);
      }
      push(`trailer\n<< /Size 6 /Root 1 0 R >>\nstartxref\n${xrefOffset}\n%%EOF`);

      const pdf = new Uint8Array(length);
      let cursor = 0;
      for (const part of parts) {
        pdf.set(part, cursor);
        cursor += part.length;
      }
      return pdf;
    },
    mealLabel(type) {
      return { breakfast: "아침", lunch: "점심", dinner: "저녁", snack: "간식" }[type] || type;
    },
    mealTypeClass(type) {
      return {
        breakfast: "meal-type-breakfast",
        lunch: "meal-type-lunch",
        dinner: "meal-type-dinner",
        snack: "meal-type-snack"
      }[type] || "meal-type-default";
    },
    goalLabel(goal) {
      return { loss: "감량", maintain: "유지", gain: "증량" }[goal] || goal;
    },
    categoryLabel(category) {
      return { meal: "식단", question: "질문", tip: "팁" }[category] || category;
    },
    categoryClass(category) {
      return {
        meal: "category-meal",
        question: "category-question",
        tip: "category-tip"
      }[category] || "category-default";
    },
    linkedMeal(post) {
      const mealId = post && post.linkedMealId;
      if (!mealId) return null;
      return this.meals.find(meal => String(meal.id) === String(mealId)) || null;
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
    toneClass(value) {
      const source = String(value || "");
      const tones = ["tone-green", "tone-mint", "tone-orange", "tone-purple", "tone-blue", "tone-rose"];
      const sum = [...source].reduce((acc, char) => acc + char.charCodeAt(0), 0);
      return tones[sum % tones.length];
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

<div v-if="route === 'login' || route === 'signup'" class="auth-page auth-page-modern">
  <div class="auth-layout">
    <section class="auth-brand-panel">
      <div class="auth-brand-content">
        <div class="eyebrow">YamYam Coach</div>
        <h1>식단 한 장으로 완성되는 데이터 분석</h1>
        <p>당신의 일상을 더 가볍고 건강하게. 나만의 페이스에 맞춘 스마트한 식단 관리를 시작해보세요.</p>

        <div class="auth-coach-card">
          <div class="auth-coach-head">
            <div class="auth-coach-icon"><i class="bi bi-stars"></i></div>
            <div>
              <span>AI Coach</span>
              <strong>식단 피드백</strong>
            </div>
          </div>
          <div class="auth-chat-preview">
            <p class="auth-chat-user">오늘 식단을 기록했어요.</p>
            <p class="auth-chat-ai">기록한 식단을 바탕으로 균형과 개선점을 정리해드릴게요.</p>
          </div>
          <div class="auth-coach-tags">
            <span>식단 기록</span>
            <span>AI 분석</span>
            <span>식단 기록</span>
          </div>
        </div>
      </div>
    </section>

    <section class="auth-panel" v-if="route === 'login'">
      <div class="auth-form-card">
        <div class="auth-mobile-brand">YamYam Coach</div>
        <div class="auth-form-header">
          <h2>로그인</h2>
          <p>계정에 로그인해 식단 기록과 AI 코칭을 확인하세요.</p>
        </div>
        <form @submit.prevent="login" class="auth-form-stack">
          <label class="auth-field">
            <span>이메일</span>
            <input v-model="loginForm.email" type="email" placeholder="hello@example.com" required>
          </label>
          <label class="auth-field">
            <span>비밀번호</span>
            <input v-model="loginForm.password" type="password" placeholder="비밀번호" required>
          </label>
          <button class="auth-primary-btn" type="submit">로그인</button>
          <button type="button" class="auth-secondary-btn" @click="go('/auth/signup')">회원가입</button>
        </form>
      </div>
    </section>

    <section class="auth-panel" v-else>
      <div class="auth-form-card auth-signup-card">
        <div class="auth-mobile-brand">YamYam Coach</div>
        <div class="auth-form-header">
          <h2>회원가입</h2>
          <p>건강한 여정을 시작하기 위해 정보를 입력해 주세요.</p>
        </div>
        <form @submit.prevent="signup" class="auth-form-stack">
          <label class="auth-field">
            <span>이메일</span>
            <input v-model="signupForm.email" type="email" placeholder="hello@example.com" required>
          </label>
          <label class="auth-field">
            <span>비밀번호</span>
            <input v-model="signupForm.password" type="password" placeholder="비밀번호" required>
          </label>
          <label class="auth-field">
            <span>닉네임</span>
            <input v-model="signupForm.nickname" placeholder="야미야미" required>
          </label>

          <div class="auth-form-grid">
            <label class="auth-field">
              <span>성별</span>
              <select v-model="signupForm.gender">
                <option value="female">여성</option>
                <option value="male">남성</option>
              </select>
            </label>
            <label class="auth-field">
              <span>출생연도</span>
              <input v-model.number="signupForm.birthYear" type="number" placeholder="YYYY">
            </label>
          </div>

          <div class="auth-form-grid">
            <label class="auth-field auth-unit-field">
              <span>키</span>
              <input v-model.number="signupForm.height" type="number" step="0.1" placeholder="0">
              <em>cm</em>
            </label>
            <label class="auth-field auth-unit-field">
              <span>몸무게</span>
              <input v-model.number="signupForm.weight" type="number" step="0.1" placeholder="0">
              <em>kg</em>
            </label>
          </div>

          <div class="auth-field">
            <span>목표</span>
            <div class="auth-goal-grid" role="group" aria-label="목표">
              <button type="button" :class="['auth-goal-option', signupForm.goal === 'loss' ? 'active' : '']" @click="signupForm.goal = 'loss'">감량</button>
              <button type="button" :class="['auth-goal-option', signupForm.goal === 'maintain' ? 'active' : '']" @click="signupForm.goal = 'maintain'">유지</button>
              <button type="button" :class="['auth-goal-option', signupForm.goal === 'gain' ? 'active' : '']" @click="signupForm.goal = 'gain'">증량</button>
            </div>
          </div>

          <label class="auth-field">
            <span>건강 메모</span>
            <textarea v-model="signupForm.healthNote" rows="3" placeholder="특이사항이나 알레르기 등을 적어주세요."></textarea>
          </label>
          <button class="auth-primary-btn" type="submit">가입하고 시작</button>
          <button type="button" class="auth-secondary-btn" @click="go('/auth/login')">로그인으로 이동</button>
        </form>
      </div>
    </section>
  </div>
  <div v-if="toast" class="toast-line">{{ toast }}</div>
</div>

<div v-else-if="false && (route === 'login' || route === 'signup')" class="auth-page">
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
  <header class="app-nav">
    <div class="container app-nav-inner">
      <a class="brand-mark" href="/home" @click.prevent="go('/home')">
        <span class="brand-icon"><i class="bi bi-flower1"></i></span>
        <span>YamYam</span>
      </a>
      <nav class="desktop-nav">
        <a :class="navClass('/home')" href="/home" @click.prevent="go('/home')">Dashboard</a>
        <a :class="navClass('/meals')" href="/meals" @click.prevent="go('/meals')">History</a>
        <a :class="navClass('/reports')" href="/reports" @click.prevent="go('/reports')">Reports</a>
        <a :class="navClass('/community')" href="/community" @click.prevent="go('/community')">Community</a>
        <a :class="navClass('/coach')" href="/coach" @click.prevent="go('/coach')">AI Coach</a>
        <a :class="navClass('/coach-v2')" href="/coach-v2" @click.prevent="go('/coach-v2')">AI Coach V2</a>
        <a :class="navClass('/challenge')" href="/challenge" @click.prevent="go('/challenge')">Challenge</a>
      </nav>
      <div class="nav-actions">
        <button class="btn btn-success btn-sm" @click="go('/meals/new')">
          <i class="bi bi-plus-lg me-1"></i>새 식단
        </button>
        <button class="icon-button d-none d-sm-inline-flex" @click="go('/profile')" :title="displayName">
          <i class="bi bi-person"></i>
        </button>
        <button class="icon-button" @click="logout" title="로그아웃">
          <i class="bi bi-box-arrow-right"></i>
        </button>
      </div>
    </div>
  </header>

  <main class="container app-shell">
    <div class="loading-line" v-if="loading">불러오는 중...</div>

    <section class="hero-panel mb-4" v-if="route === 'home'">
      <div class="panel-body page-title mb-0">
        <div>
          <div class="eyebrow text-white">Dashboard</div>
          <h1>Welcome back, {{ displayName }}님</h1>
          <p class="muted mt-2 mb-0">오늘도 건강한 하루를 만들어볼까요? 식단과 코칭을 한 화면에서 확인하세요.</p>
        </div>
        <button class="btn btn-light" @click="go('/meals/new')">
          <i class="bi bi-plus-lg me-1"></i>식단 등록
        </button>
      </div>
    </section>

    <div class="page-title" v-else-if="!['coach', 'coachV2', 'challenge'].includes(route)">
      <div>
        <div class="eyebrow">YamYam</div>
        <h1>{{ pageTitle }}</h1>
      </div>
      <button v-if="route === 'meals'" class="btn btn-success" @click="go('/meals/new')">
        <i class="bi bi-plus-lg me-1"></i>새 식단
      </button>
    </div>

    <section v-if="route === 'home'">
      <div class="dashboard-grid">
        <div class="dashboard-main">
          <div class="two-grid mb-4">
            <div class="chart-panel calorie-card">
              <div class="section-head">
                <h2><i class="bi bi-fire text-accent"></i> 오늘 칼로리</h2>
              </div>
              <div class="calorie-content">
                <div class="donut" :style="{ '--pct': caloriePct }">
                  <div class="donut-inner">
                    <strong>{{ todaySummary.calories }}</strong>
                    <span>/ {{ dailyGoal.calories }} kcal</span>
                  </div>
                </div>
                <p class="muted mb-0">목표까지 <strong class="text-primary">{{ calorieRemaining }} kcal</strong> 남았어요.</p>
              </div>
            </div>

            <div class="chart-panel macro-card">
              <div class="section-head">
                <h2><i class="bi bi-pie-chart text-primary"></i> 탄단지 비율</h2>
              </div>
              <div class="macro-list">
                <div>
                  <div class="macro-row"><span>탄수화물</span><span>{{ Math.round(todaySummary.carbs) }}g · {{ todaySummary.carbsPct }}%</span></div>
                  <div class="bar-track"><div class="macro-carb" :style="{ width: todaySummary.carbsPct + '%' }"></div></div>
                </div>
                <div>
                  <div class="macro-row"><span>단백질</span><span>{{ Math.round(todaySummary.protein) }}g · {{ todaySummary.proteinPct }}%</span></div>
                  <div class="bar-track"><div class="macro-protein" :style="{ width: todaySummary.proteinPct + '%' }"></div></div>
                </div>
                <div>
                  <div class="macro-row"><span>지방</span><span>{{ Math.round(todaySummary.fat) }}g · {{ todaySummary.fatPct }}%</span></div>
                  <div class="bar-track"><div class="macro-fat" :style="{ width: todaySummary.fatPct + '%' }"></div></div>
                </div>
              </div>
              <div class="insight-box mt-3"><i class="bi bi-lightbulb me-2 text-primary"></i>{{ macroTip }}</div>
            </div>
          </div>

          <div class="panel">
            <div class="panel-body">
              <div class="section-head">
                <h2><i class="bi bi-egg-fried text-primary"></i> 최근 식단 기록</h2>
                <button class="btn btn-outline-success btn-sm" @click="go('/meals')">전체 보기</button>
              </div>
              <div v-if="!filteredMeals.length" class="empty-state">
                <i class="bi bi-plus-circle display-6 d-block mb-2"></i>
                아직 기록된 식단이 없습니다.
                <div class="mt-3"><button class="btn btn-success btn-sm" @click="go('/meals/new')">첫 식단 기록하기</button></div>
              </div>
              <a v-for="meal in filteredMeals.slice(0, 5)" :key="meal.id" class="meal-row meal-card-row" href="#" @click.prevent="go('/meals/detail?mealId=' + meal.id)">
                <span :class="['meal-type-box', mealTypeClass(meal.mealType)]">{{ mealLabel(meal.mealType) }}</span>
                <div>
                  <strong>{{ mealFoodNames(meal) }}</strong>
                  <div class="muted">{{ meal.mealDate }} · {{ (meal.foods || []).length }}개 음식</div>
                </div>
                <div class="text-end">
                  <strong>{{ mealCalories(meal) }} kcal</strong>
                  <div class="mini-chip-list mt-1">
                    <span>C {{ Math.round(meal.nutrition && meal.nutrition.carbs || 0) }}</span>
                    <span>P {{ Math.round(meal.nutrition && meal.nutrition.protein || 0) }}</span>
                    <span>F {{ Math.round(meal.nutrition && meal.nutrition.fat || 0) }}</span>
                  </div>
                </div>
              </a>
              <button class="add-meal-row" @click="go('/meals/new')">
                <i class="bi bi-plus-circle"></i>
                식사 기록하기
              </button>
            </div>
          </div>
        </div>

        <aside class="dashboard-side">
          <div class="panel">
            <div class="panel-body">
              <div class="section-head"><h2><i class="bi bi-trophy text-accent"></i> 진행 중인 챌린지</h2></div>
              <div v-if="!joinedChallenges.length" class="empty-state compact-empty">
                아직 참여 중인 챌린지가 없어요.
                <div class="mt-3"><button class="btn btn-outline-success btn-sm" @click="go('/challenge')">챌린지 둘러보기</button></div>
              </div>
              <a v-for="challenge in joinedChallenges.slice(0, 3)" :key="challenge.id" class="insight-box d-block mb-2" href="#" @click.prevent="go('/challenge')">
                <strong>{{ challenge.title }}</strong>
                <div class="muted mt-1">{{ challenge.description }}</div>
                <template v-if="memberships[challenge.id]">
                  <div class="bar-track mt-3 mb-2">
                    <div class="bar-fill" :style="{ width: Math.min(100, memberships[challenge.id].progress * 100 / challenge.targetCount) + '%' }"></div>
                  </div>
                  <div class="muted">{{ memberships[challenge.id].progress }} / {{ challenge.targetCount }}</div>
                </template>
              </a>
            </div>
          </div>

          <div class="coach-teaser">
            <div class="d-flex align-items-center gap-2 mb-2">
              <i class="bi bi-robot"></i>
              <strong>AI 코치 한마디</strong>
            </div>
            <p class="mb-0">"최근 식단을 바탕으로 다음 식사의 균형을 제안해드릴게요."</p>
          </div>
        </aside>
      </div>
    </section>

    <section v-if="route === 'meals'" class="meals-page-grid">
      <div class="meals-main">
        <div class="panel filter-panel mb-4">
          <div class="panel-body">
            <div class="row g-3 align-items-end">
              <div class="col-md-6 col-xl-3">
                <label class="form-label">기간 설정</label>
                <input class="form-control" type="date" v-model="mealFilters.startDate">
              </div>
              <div class="col-md-6 col-xl-3">
                <label class="form-label">종료일</label>
                <input class="form-control" type="date" v-model="mealFilters.endDate">
              </div>
              <div class="col-md-6 col-xl-3">
                <label class="form-label">식사 유형</label>
                <select class="form-select" v-model="mealFilters.mealType">
                  <option value="">전체보기</option>
                  <option value="breakfast">아침</option>
                  <option value="lunch">점심</option>
                  <option value="dinner">저녁</option>
                  <option value="snack">간식</option>
                </select>
              </div>
              <div class="col-md-6 col-xl-3">
                <label class="form-label">정렬 기준</label>
                <select class="form-select" v-model="mealFilters.sortKey">
                  <option value="dateDesc">최신순</option>
                  <option value="dateAsc">오래된순</option>
                  <option value="energyDesc">칼로리 높은순</option>
                </select>
              </div>
            </div>
          </div>
        </div>

        <div class="meal-log-list">
          <div v-if="!filteredMeals.length" class="panel">
            <div class="empty-state">
              <i class="bi bi-journal-plus display-6 d-block mb-2"></i>
              조회된 식단이 없습니다.
              <div class="mt-3"><button class="btn btn-success btn-sm" @click="go('/meals/new')">새 식단 등록</button></div>
            </div>
          </div>

          <article v-for="meal in filteredMeals" :key="meal.id" class="meal-log-card">
            <a class="meal-log-card-main" href="#" @click.prevent="go('/meals/detail?mealId=' + meal.id)">
              <div class="meal-log-meta">
                <strong>{{ meal.mealDate }}</strong>
                <span :class="['badge-soft', 'meal-type-badge', mealTypeClass(meal.mealType)]">{{ mealLabel(meal.mealType) }}</span>
              </div>
              <div class="meal-log-content">
                <div class="food-chip-list">
                  <span v-for="food in (meal.foods || []).slice(0, 4)" :key="food.code || food.foodCode || food.name" class="food-chip">{{ food.name }}</span>
                  <span v-if="(meal.foods || []).length > 4" class="food-chip">+{{ (meal.foods || []).length - 4 }}</span>
                  <span v-if="!(meal.foods || []).length" class="food-chip">음식 정보 없음</span>
                </div>
                <p class="muted fst-italic mb-0">{{ meal.memo || "메모 없음" }}</p>
              </div>
              <div class="meal-log-calories">
                <strong>{{ mealCalories(meal) }} kcal</strong>
                <span>{{ (meal.foods || []).length }}개 음식</span>
              </div>
            </a>
            <div class="meal-log-actions">
              <button class="icon-button" @click.stop.prevent="go('/meals/edit?mealId=' + meal.id)" title="수정">
                <i class="bi bi-pencil"></i>
              </button>
              <button class="icon-button danger" @click.stop.prevent="deleteMeal(meal.id)" title="삭제">
                <i class="bi bi-trash"></i>
              </button>
            </div>
          </article>
        </div>
      </div>

      <aside class="meals-summary">
        <div class="panel sticky-summary">
          <div class="panel-body">
            <div class="section-head">
              <h2><i class="bi bi-graph-up text-primary"></i> 기간 요약</h2>
            </div>
            <div class="summary-stack">
              <div class="summary-row">
                <span>총 식사 횟수</span>
                <strong>{{ filteredMealSummary.count }}회</strong>
              </div>
              <div class="summary-row">
                <span>총 칼로리</span>
                <strong>{{ filteredMealSummary.calories }} kcal</strong>
              </div>
              <div class="summary-row">
                <span>평균 칼로리</span>
                <strong>{{ filteredMealSummary.avgCalories }} kcal</strong>
              </div>
            </div>
            <div class="macro-summary">
              <div><span>탄수화물</span><strong>{{ filteredMealSummary.carbs }}g</strong></div>
              <div><span>단백질</span><strong>{{ filteredMealSummary.protein }}g</strong></div>
              <div><span>지방</span><strong>{{ filteredMealSummary.fat }}g</strong></div>
            </div>
          </div>
        </div>
      </aside>
    </section>

    <section v-if="route === 'mealForm'" class="meal-form-grid">
      <form class="meal-form-left" @submit.prevent="saveMeal">
        <div class="panel meal-info-panel">
          <div class="panel-body">
            <div class="section-head">
              <h2><i class="bi bi-info-circle text-primary"></i> 기본 정보</h2>
            </div>
            <div class="d-grid gap-3">
              <div>
                <label class="form-label">날짜</label>
                <input class="form-control soft-input" v-model="mealForm.mealDate" type="date" required>
              </div>
              <div>
                <label class="form-label">식사 종류</label>
                <div class="meal-type-selector">
                  <button type="button" :class="['meal-type-choice', mealForm.mealType === 'breakfast' && 'active', 'meal-type-breakfast']" @click="mealForm.mealType = 'breakfast'">아침</button>
                  <button type="button" :class="['meal-type-choice', mealForm.mealType === 'lunch' && 'active', 'meal-type-lunch']" @click="mealForm.mealType = 'lunch'">점심</button>
                  <button type="button" :class="['meal-type-choice', mealForm.mealType === 'dinner' && 'active', 'meal-type-dinner']" @click="mealForm.mealType = 'dinner'">저녁</button>
                  <button type="button" :class="['meal-type-choice', mealForm.mealType === 'snack' && 'active', 'meal-type-snack']" @click="mealForm.mealType = 'snack'">간식</button>
                </div>
              </div>
              <div>
                <label class="form-label">메모</label>
                <textarea class="form-control soft-input" v-model="mealForm.memo" rows="3" placeholder="오늘 식사에 대한 메모를 남겨보세요."></textarea>
              </div>
            </div>
          </div>
        </div>

        <div class="panel selected-food-panel">
          <div class="panel-body">
            <div class="section-head">
              <h2><i class="bi bi-card-checklist text-primary"></i> 선택한 음식</h2>
              <span class="badge-soft">{{ mealForm.foods.length }}개 항목</span>
            </div>
            <div v-if="!mealForm.foods.length" class="empty-state">
              음식 검색 결과에서 추가할 음식을 선택하세요.
            </div>
            <div v-for="food in mealForm.foods" :key="food.foodCode" class="selected-food-row">
              <div>
                <strong>{{ food.name }}</strong>
                <div class="muted">{{ food.category }} · 100g당 {{ Math.round(food.energy || 0) }} kcal</div>
              </div>
              <div class="selected-food-controls">
                <div class="gram-input">
                  <input v-model.number="food.grams" type="number" step="1" min="1">
                  <span>g</span>
                </div>
                <button type="button" class="icon-button danger" @click="removeFood(food.foodCode)" title="제거">
                  <i class="bi bi-x-lg"></i>
                </button>
              </div>
            </div>

            <div class="selected-summary">
              <div class="summary-total">
                <span>총 칼로리</span>
                <strong>{{ mealFormSummary.calories }} kcal</strong>
              </div>
              <div class="macro-summary compact">
                <div><span>탄수화물</span><strong>{{ mealFormSummary.carbs }}g</strong></div>
                <div><span>단백질</span><strong>{{ mealFormSummary.protein }}g</strong></div>
                <div><span>지방</span><strong>{{ mealFormSummary.fat }}g</strong></div>
              </div>
            </div>

            <button class="btn btn-success btn-lg w-100 mt-4">
              <i class="bi bi-check-circle me-1"></i>{{ mealForm.id ? "식단 수정 완료" : "식단 등록 완료" }}
            </button>
          </div>
        </div>
      </form>

      <div class="panel food-search-panel">
        <div class="panel-body">
          <div class="section-head">
            <div>
              <h2><i class="bi bi-search text-primary"></i> 음식 검색</h2>
              <p class="muted mb-0">데이터베이스에서 음식을 검색하여 추가하세요.</p>
            </div>
          </div>
          <div class="input-group search-input mb-3">
            <span class="input-group-text"><i class="bi bi-search"></i></span>
            <input class="form-control" v-model="foodKeyword" placeholder="예: 연어, 고구마, 오트밀">
            <button class="btn btn-outline-success" @click="loadFoods">검색</button>
          </div>

          <div class="food-result-list">
            <div v-if="!foods.length" class="empty-state">검색 결과가 없습니다.</div>
            <div v-for="food in foods" :key="food.code" class="food-result-row">
              <div :class="['food-result-icon', toneClass(food.code || food.name)]">{{ String(food.name || food.category || "?").slice(0, 2) }}</div>
              <div class="food-result-body">
                <strong>{{ food.name }}</strong>
                <div class="muted">
                  {{ food.category }} · 100g당 {{ Math.round(food.energy || 0) }} kcal
                  · 탄 {{ Math.round(food.carbs || 0) }}g
                  · 단 {{ Math.round(food.protein || 0) }}g
                  · 지 {{ Math.round(food.fat || 0) }}g
                </div>
              </div>
              <button class="add-food-button" @click="addFood(food)" title="추가">
                <i class="bi bi-plus-lg"></i>
              </button>
            </div>
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

    <section v-if="route === 'profile'" class="profile-page">
      <form class="profile-card" @submit.prevent="saveProfile">
        <section class="profile-header">
          <div class="profile-avatar">{{ displayName.slice(0, 1) }}</div>
          <div class="profile-header-copy">
            <h2>{{ displayName }}님</h2>
            <p>{{ profileForm.email }}</p>
            <span class="badge-soft">{{ goalLabel(profileForm.goal) }}</span>
          </div>
        </section>

        <div class="profile-form-grid">
          <section class="profile-section">
            <div class="section-title-inline">
              <i class="bi bi-person text-primary"></i>
              <h3>기본 정보</h3>
            </div>
            <div class="d-grid gap-3">
              <div>
                <label class="form-label">이메일</label>
                <input class="form-control soft-input" v-model="profileForm.email" disabled>
              </div>
              <div>
                <label class="form-label">닉네임</label>
                <input class="form-control soft-input" v-model="profileForm.nickname" required>
              </div>
              <div>
                <label class="form-label">성별</label>
                <select class="form-select soft-input" v-model="profileForm.gender">
                  <option value="female">여성</option>
                  <option value="male">남성</option>
                </select>
              </div>
              <div>
                <label class="form-label">출생연도</label>
                <input class="form-control soft-input" v-model.number="profileForm.birthYear" type="number">
              </div>
              <div class="profile-two-cols">
                <div>
                  <label class="form-label">키 (cm)</label>
                  <input class="form-control soft-input" v-model.number="profileForm.height" type="number" step="0.1">
                </div>
                <div>
                  <label class="form-label">몸무게 (kg)</label>
                  <input class="form-control soft-input" v-model.number="profileForm.weight" type="number" step="0.1">
                </div>
              </div>
            </div>
          </section>

          <section class="profile-section">
            <div class="section-title-inline">
              <i class="bi bi-flag text-primary"></i>
              <h3>건강 목표</h3>
            </div>
            <div class="goal-choice-list">
              <label :class="['goal-choice', profileForm.goal === 'loss' && 'active']">
                <input class="visually-hidden" type="radio" value="loss" v-model="profileForm.goal">
                <span class="goal-icon"><i class="bi bi-graph-down-arrow"></i></span>
                <span>체중 감량</span>
              </label>
              <label :class="['goal-choice', profileForm.goal === 'maintain' && 'active']">
                <input class="visually-hidden" type="radio" value="maintain" v-model="profileForm.goal">
                <span class="goal-icon"><i class="bi bi-arrow-repeat"></i></span>
                <span>현재 유지</span>
              </label>
              <label :class="['goal-choice', profileForm.goal === 'gain' && 'active']">
                <input class="visually-hidden" type="radio" value="gain" v-model="profileForm.goal">
                <span class="goal-icon"><i class="bi bi-lightning-charge"></i></span>
                <span>체중 증량</span>
              </label>
            </div>
            <div class="mt-3">
              <label class="form-label">건강 메모</label>
              <textarea class="form-control soft-input" v-model="profileForm.healthNote" rows="4" placeholder="특별한 식이요법이나 주의사항이 있다면 기록해주세요."></textarea>
            </div>
          </section>
        </div>

        <section class="profile-actions">
          <button type="button" class="btn btn-outline-danger" @click="deactivate">
            <i class="bi bi-person-x me-1"></i>계정 비활성화
          </button>
          <button class="btn btn-success">
            <i class="bi bi-save me-1"></i>변경사항 저장
          </button>
        </section>
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

    <section v-if="route === 'challenge'" class="challenge-page">
      <div class="challenge-main">
        <div class="challenge-header">
          <div>
            <div class="eyebrow">Health Challenges</div>
            <h2>건강 챌린지</h2>
            <p>함께해서 더 즐거운 건강한 목표 달성</p>
          </div>
          <button class="btn btn-success" @click="challengeCreateOpen = !challengeCreateOpen">
            <i class="bi bi-plus-lg me-1"></i>새 챌린지 만들기
          </button>
        </div>

        <section class="challenge-section">
          <div class="challenge-section-title">
            <i class="bi bi-fire"></i>
            <h3>진행 중인 챌린지</h3>
          </div>
          <div v-if="!joinedChallenges.length" class="challenge-empty">
            아직 참여 중인 챌린지가 없습니다. 아래 추천 챌린지에서 참여할 수 있습니다.
          </div>
          <div v-else class="active-challenge-grid">
            <article v-for="challenge in joinedChallenges" :key="challenge.id" class="active-challenge-card">
              <div class="challenge-card-head">
                <div class="challenge-title-row">
                  <span class="challenge-icon"><i :class="['bi', challengeIconClass(challenge.category)]"></i></span>
                  <div>
                    <h4>{{ challenge.title }}</h4>
                    <p>{{ challenge.description }}</p>
                  </div>
                </div>
                <span class="challenge-dday">{{ challengeDday(challenge) }}</span>
              </div>
              <div class="challenge-progress-meta">
                <span>진행률 {{ challengePercent(challenge) }}%</span>
                <span>{{ challengeProgress(challenge) }} / {{ challenge.targetCount }}</span>
              </div>
              <div class="bar-track challenge-progress">
                <div class="bar-fill" :style="{ width: challengePercent(challenge) + '%' }"></div>
              </div>
              <input class="form-range challenge-range" type="range" min="0" :max="challenge.targetCount" :value="challengeProgress(challenge)" @change="updateChallengeProgress(challenge.id, $event.target.value)">
            </article>
          </div>
        </section>

        <section class="challenge-section">
          <div class="challenge-section-title justify-content-between">
            <div class="d-flex align-items-center gap-2">
              <i class="bi bi-stars"></i>
              <h3>추천 챌린지</h3>
            </div>
            <span class="muted">{{ availableChallenges.length }}개</span>
          </div>
          <div v-if="!challenges.length" class="challenge-empty">등록된 챌린지가 없습니다.</div>
          <div v-else-if="!availableChallenges.length" class="challenge-empty">참여 가능한 새 챌린지가 없습니다.</div>
          <div v-else class="challenge-card-grid">
            <article v-for="challenge in availableChallenges" :key="challenge.id" class="challenge-card">
              <div class="challenge-image" :class="['challenge-image-' + (challenge.category || 'default')]">
                <i :class="['bi', challengeIconClass(challenge.category)]"></i>
                <span><i class="bi bi-people"></i>{{ challengeParticipantCount(challenge) }}명</span>
              </div>
              <div class="challenge-card-body">
                <span class="badge-soft">{{ challenge.category || '챌린지' }}</span>
                <h4>{{ challenge.title }}</h4>
                <p>{{ challenge.description }}</p>
                <div class="challenge-card-foot">
                  <span>{{ challengeDday(challenge) }}</span>
                  <span>목표 {{ challenge.targetCount }}</span>
                </div>
                <button class="btn btn-outline-success w-100" @click="joinChallenge(challenge.id)">참여하기</button>
              </div>
            </article>
          </div>
        </section>
      </div>

      <aside class="challenge-side">
        <form class="panel challenge-create-panel" @submit.prevent="createChallenge" v-show="challengeCreateOpen || !challenges.length">
          <div class="panel-body d-grid gap-3">
            <h2 class="section-subtitle">챌린지 생성</h2>
            <input class="form-control soft-input" v-model="challengeForm.title" placeholder="제목" required>
            <textarea class="form-control soft-input" v-model="challengeForm.description" rows="3" placeholder="설명"></textarea>
            <input class="form-control soft-input" v-model="challengeForm.category" placeholder="분류">
            <div>
              <label class="form-label">목표 횟수</label>
              <input class="form-control soft-input" v-model.number="challengeForm.targetCount" type="number" min="1" placeholder="예: 7">
            </div>
            <input class="form-control soft-input" v-model="challengeForm.endDate" type="date" required>
            <button class="btn btn-success">생성</button>
          </div>
        </form>

        <div class="panel">
          <div class="panel-body">
            <h2 class="section-subtitle">내 챌린지 요약</h2>
            <div class="challenge-stat-list">
              <div><span>참여 중</span><strong>{{ joinedChallenges.length }}</strong></div>
              <div><span>참여 가능</span><strong>{{ availableChallenges.length }}</strong></div>
              <div><span>전체</span><strong>{{ challenges.length }}</strong></div>
            </div>
          </div>
        </div>
      </aside>
    </section>

    <section v-if="route === 'community'" class="community-page">
      <div class="community-main">
        <div class="community-filter-row">
          <button type="button" :class="['community-filter-chip', community.category === 'all' && 'active']" @click="community.category = 'all'; loadCommunity()">전체</button>
          <button type="button" :class="['community-filter-chip', community.category === 'meal' && 'active']" @click="community.category = 'meal'; loadCommunity()">식단공유</button>
          <button type="button" :class="['community-filter-chip', community.category === 'question' && 'active']" @click="community.category = 'question'; loadCommunity()">질문</button>
          <button type="button" :class="['community-filter-chip', community.category === 'tip' && 'active']" @click="community.category = 'tip'; loadCommunity()">팁</button>
        </div>

        <form class="community-compose" @submit.prevent="createPost">
          <div :class="['avatar', 'compact', toneClass(displayName)]">{{ displayName.slice(0, 1) }}</div>
          <div class="community-compose-body">
            <div class="compose-grid">
              <select class="form-select soft-input" v-model="postForm.category">
                <option value="meal">식단</option>
                <option value="question">질문</option>
                <option value="tip">팁</option>
              </select>
              <select class="form-select soft-input" v-model="postForm.linkedMealId">
                <option value="">연결 식단 없음</option>
                <option v-for="meal in meals" :key="meal.id" :value="meal.id">{{ meal.mealDate }} {{ mealLabel(meal.mealType) }}</option>
              </select>
            </div>
            <input class="form-control soft-input" v-model="postForm.title" placeholder="제목" required>
            <textarea class="form-control soft-input" v-model="postForm.content" rows="3" placeholder="오늘 어떤 건강한 이야기를 나누고 싶으신가요?" required></textarea>
            <div class="d-flex justify-content-end">
              <button class="btn btn-success">
                <i class="bi bi-send me-1"></i>게시
              </button>
            </div>
          </div>
        </form>

        <div class="community-feed">
          <div v-if="!community.posts.length" class="panel">
            <div class="empty-state">게시글이 없습니다.</div>
          </div>
          <article v-for="post in community.posts" :key="post.id" class="community-post-card">
            <div class="community-post-head">
              <div class="d-flex align-items-center gap-2">
                <span :class="['avatar', 'compact', toneClass(authorName(post.userId))]">{{ authorName(post.userId).slice(0, 1) }}</span>
                <div>
                  <strong>{{ authorName(post.userId) }}</strong>
                  <div class="muted">{{ fmt(post.createdAt) }}</div>
                </div>
              </div>
              <span :class="['badge-soft', 'community-category-badge', categoryClass(post.category)]">{{ categoryLabel(post.category) }}</span>
            </div>
            <h2>{{ post.title }}</h2>
            <p class="community-post-content">{{ post.content }}</p>

            <div v-if="linkedMeal(post)" class="linked-meal-card">
              <div class="linked-meal-icon"><i class="bi bi-egg-fried"></i></div>
              <div>
                <strong>{{ mealFoodNames(linkedMeal(post)) }}</strong>
                <div class="muted">{{ linkedMeal(post).mealDate }} · {{ mealLabel(linkedMeal(post).mealType) }}</div>
              </div>
              <strong class="text-primary">{{ mealCalories(linkedMeal(post)) }} kcal</strong>
            </div>

            <div class="comment-list" v-if="(community.comments[post.id] || []).length">
              <div class="comment-row" v-for="comment in community.comments[post.id] || []" :key="comment.id">
                <span :class="['avatar', 'mini', toneClass(authorName(comment.userId))]">{{ authorName(comment.userId).slice(0, 1) }}</span>
                <div>
                  <strong>{{ authorName(comment.userId) }}</strong>
                  <p>{{ comment.content }}</p>
                </div>
              </div>
            </div>

            <form class="comment-form" @submit.prevent="createComment(post.id)">
              <input class="form-control" v-model="commentDrafts[post.id]" placeholder="댓글을 입력하세요">
              <button class="btn btn-outline-success">등록</button>
            </form>
          </article>
        </div>
      </div>

      <aside class="community-side">
        <div class="panel">
          <div class="panel-body">
            <div class="section-head"><h2><i class="bi bi-info-circle text-primary"></i> 커뮤니티</h2></div>
            <p class="muted mb-0">식단 공유, 질문, 팁 게시글을 확인하고 댓글로 의견을 나눌 수 있습니다.</p>
          </div>
        </div>
        <div class="panel">
          <div class="panel-body">
            <div class="section-head"><h2><i class="bi bi-link-45deg text-primary"></i> 연결 가능 식단</h2></div>
            <div v-if="!meals.length" class="empty-state compact-empty">연결할 식단이 없습니다.</div>
            <div v-for="meal in filteredMeals.slice(0, 3)" :key="meal.id" class="side-meal-row">
              <span :class="['badge-soft', 'meal-type-badge', mealTypeClass(meal.mealType)]">{{ mealLabel(meal.mealType) }}</span>
              <div>
                <strong>{{ meal.mealDate }}</strong>
                <div class="muted">{{ mealCalories(meal) }} kcal</div>
              </div>
            </div>
          </div>
        </div>
      </aside>
    </section>

    <section v-if="route === 'reports'" class="reports-page">
      <div class="reports-main">
        <div class="panel report-builder-panel">
          <div class="panel-body">
            <div class="section-head">
              <h2><i class="bi bi-file-earmark-pdf text-primary"></i> 일일 식단 리포트</h2>
            </div>
            <form class="report-form" @submit.prevent="generateDailyReport">
              <div>
                <label class="form-label">리포트 날짜</label>
                <input class="form-control soft-input" type="date" v-model="reportForm.date" required>
              </div>
              <button class="btn btn-success" :disabled="reportLoading">
                <span v-if="reportLoading" class="spinner-border spinner-border-sm me-2"></span>
                {{ reportLoading ? "생성 중..." : "리포트 생성" }}
                <i v-if="!reportLoading" class="bi bi-download ms-1"></i>
              </button>
            </form>
          </div>
        </div>

        <div class="panel">
          <div class="panel-body">
            <div class="section-head">
              <h2><i class="bi bi-calendar-check text-primary"></i> 선택 날짜 식단</h2>
              <span class="badge-soft">{{ reportMeals().length }}건</span>
            </div>
            <div v-if="!reportMeals().length" class="empty-state compact-empty">
              선택한 날짜에 식단 기록이 없습니다.
            </div>
            <div v-else class="report-meal-list">
              <div v-for="meal in reportMeals()" :key="meal.id" class="report-meal-row">
                <span :class="['badge-soft', 'meal-type-badge', mealTypeClass(meal.mealType)]">{{ mealLabel(meal.mealType) }}</span>
                <div>
                  <strong>{{ mealFoodNames(meal) }}</strong>
                  <div class="muted">{{ mealCalories(meal) }} kcal · 탄 {{ Number(meal.nutrition && meal.nutrition.carbs || 0).toFixed(1) }}g · 단 {{ Number(meal.nutrition && meal.nutrition.protein || 0).toFixed(1) }}g · 지 {{ Number(meal.nutrition && meal.nutrition.fat || 0).toFixed(1) }}g</div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <aside class="reports-side">
        <div class="panel">
          <div class="panel-body">
            <div class="section-head"><h2><i class="bi bi-pie-chart text-primary"></i> 영양 합계</h2></div>
            <div class="report-total-list">
              <div><span>칼로리</span><strong>{{ reportSummary(reportMeals()).calories.toFixed(1) }} kcal</strong></div>
              <div><span>탄수화물</span><strong>{{ reportSummary(reportMeals()).carbs.toFixed(1) }} g</strong></div>
              <div><span>단백질</span><strong>{{ reportSummary(reportMeals()).protein.toFixed(1) }} g</strong></div>
              <div><span>지방</span><strong>{{ reportSummary(reportMeals()).fat.toFixed(1) }} g</strong></div>
            </div>
          </div>
        </div>
      </aside>
    </section>

    <section v-if="route === 'coach'" class="coach-page">
      <aside class="coach-input-stack">
        <div class="coach-page-copy">
          <h2>AI 코치</h2>
          <p>선택한 식단 또는 최근 식단을 바탕으로 개선 조언을 요청합니다.</p>
          <div class="coach-version-tabs">
            <button type="button" class="active" @click="go('/coach')">V1 기본 코치</button>
            <button type="button" @click="go('/coach-v2')">V2 도구 연동</button>
          </div>
        </div>

        <form class="coach-card" @submit.prevent="askCoach">
          <div class="section-head">
            <h2><i class="bi bi-sliders text-primary"></i> 질문 설정</h2>
          </div>
          <div class="d-grid gap-3">
            <div>
              <label class="form-label">컨텍스트 식단</label>
              <select class="form-select soft-input" v-model="coachForm.mealId">
                <option value="">최근 식단 전체</option>
                <option v-for="meal in meals" :key="meal.id" :value="meal.id">{{ meal.mealDate }} {{ mealLabel(meal.mealType) }}</option>
              </select>
            </div>
            <div>
              <label class="form-label">질문</label>
              <textarea class="form-control soft-input coach-question" v-model="coachForm.question" rows="6" placeholder="예: 최근 식단에서 개선할 점을 알려줘."></textarea>
            </div>
            <button class="btn btn-success btn-lg" :disabled="coachLoading">
              <span v-if="coachLoading" class="spinner-border spinner-border-sm me-2"></span>
              {{ coachLoading ? "분석 중..." : "조언 받기" }} <i v-if="!coachLoading" class="bi bi-send ms-1"></i>
            </button>
          </div>
        </form>
      </aside>

      <div class="coach-response-stack">
        <div class="coach-loading-card" v-if="coachLoading">
          <span class="spinner-border text-success"></span>
          <h3>코치가 식단을 분석 중입니다</h3>
          <p>요청한 식단과 질문을 바탕으로 조언을 생성하고 있어요.</p>
        </div>
        <div class="coach-response-card" v-else-if="advice">
          <div class="coach-response-head">
            <span class="coach-bot-icon"><i class="bi bi-robot"></i></span>
            <div>
              <h3>YamYam 코치</h3>
              <div class="d-flex flex-wrap gap-2 mt-1">
                <span class="badge-soft">{{ advice.provider }}</span>
                <span class="badge-soft badge-accent" v-if="advice.springAiUsed">Spring AI</span>
              </div>
            </div>
          </div>
          <div class="coach-answer">{{ advice.advice }}</div>
        </div>
        <div class="coach-empty" v-else>
          <i class="bi bi-stars"></i>
          <h3>코칭 응답 대기 중</h3>
          <p>질문을 보내면 식단 기반 조언이 이곳에 표시됩니다.</p>
        </div>
      </div>
    </section>

    <section v-if="route === 'coachV2'" class="coach-page coach-page-v2">
      <aside class="coach-input-stack">
        <div class="coach-page-copy">
          <h2>AI 코치 V2</h2>
          <p>날짜, 도시, 실행 모드를 함께 전달해 도구 연동 코칭을 요청합니다.</p>
          <div class="coach-version-tabs">
            <button type="button" @click="go('/coach')">V1 기본 코치</button>
            <button type="button" class="active" @click="go('/coach-v2')">V2 도구 연동</button>
          </div>
        </div>

        <form class="coach-card" @submit.prevent="askCoachV2">
          <div class="section-head">
            <h2><i class="bi bi-toggles text-primary"></i> 코치 설정</h2>
          </div>
          <div class="row g-3">
            <div class="col-md-6">
              <label class="form-label">분석 기준일</label>
              <input class="form-control soft-input" v-model="coachV2Form.date" type="date">
            </div>
            <div class="col-md-6">
              <label class="form-label">도시</label>
              <input class="form-control soft-input" v-model="coachV2Form.city" placeholder="Seoul">
            </div>
          </div>
          <div class="mt-3">
            <label class="form-label">실행 모드</label>
            <select class="form-select soft-input" v-model="coachV2Form.mode">
              <option value="multi">다중 Tool 연쇄</option>
              <option value="llm-driven">LLM 주도 Tool 선택</option>
            </select>
          </div>
          <div class="mt-3">
            <label class="form-label">질문</label>
            <textarea class="form-control soft-input coach-question" v-model="coachV2Form.question" rows="6" placeholder="예: 오늘 날씨와 내 식단을 바탕으로 저녁 식사와 운동을 추천해줘."></textarea>
          </div>
          <button class="btn btn-success btn-lg w-100 mt-3" :disabled="coachV2Loading">
            <span v-if="coachV2Loading" class="spinner-border spinner-border-sm me-2"></span>
            {{ coachV2Loading ? "코칭 진행 중..." : "코칭 요청" }} <i v-if="!coachV2Loading" class="bi bi-send ms-1"></i>
          </button>
        </form>
      </aside>

      <div class="coach-response-stack">
        <div class="coach-process-strip" v-if="coachV2Loading || adviceV2">
          <span><i class="bi bi-check-circle"></i> 요청 완료</span>
          <span><i :class="coachV2Loading ? 'bi bi-arrow-repeat' : 'bi bi-check-circle'"></i> {{ coachV2Form.mode }}</span>
          <span><i class="bi bi-stars"></i> {{ coachV2Loading ? "응답 생성 중" : "응답 생성 완료" }}</span>
        </div>
        <div class="coach-loading-card" v-if="coachV2Loading">
          <span class="spinner-border text-success"></span>
          <h3>V2 코칭을 진행 중입니다</h3>
          <p>날짜, 도시, 실행 모드를 바탕으로 도구 연동 결과를 생성하고 있어요.</p>
        </div>
        <div class="coach-response-card" v-else-if="adviceV2">
          <div class="coach-response-head">
            <span class="coach-bot-icon"><i class="bi bi-cpu"></i></span>
            <div>
              <h3>YamYam 코치 V2</h3>
              <div class="d-flex flex-wrap gap-2 mt-1">
                <span class="badge-soft badge-accent">{{ adviceV2.mode }}</span>
                <span class="badge-soft">{{ adviceV2.provider }}</span>
                <span class="badge-soft" v-if="adviceV2.springAiUsed">Spring AI</span>
              </div>
            </div>
          </div>
          <div class="coach-answer mb-3">{{ adviceV2.answer }}</div>
          <div v-if="adviceV2Plan.length" class="coach-subsection">
            <h4>실행 계획</h4>
            <ol>
              <li v-for="step in adviceV2Plan" :key="step">{{ step }}</li>
            </ol>
          </div>
          <div v-if="adviceV2ToolResults.length" class="coach-subsection">
            <h4>Tool 실행 결과</h4>
            <div v-for="tool in adviceV2ToolResults" :key="tool.toolName + tool.summary" class="tool-result-card">
              <div class="d-flex justify-content-between gap-2">
                <strong>{{ tool.toolName }}</strong>
                <span :class="tool.success ? 'badge-soft badge-accent' : 'badge-soft badge-danger-soft'">{{ tool.success ? '성공' : '실패' }}</span>
              </div>
              <div class="muted mt-1">{{ tool.summary }}</div>
            </div>
          </div>
        </div>
        <div class="coach-empty" v-else>
          <i class="bi bi-magic"></i>
          <h3>V2 코칭 응답 대기 중</h3>
          <p>코칭 요청을 보내면 결과와 실행 정보가 이곳에 표시됩니다.</p>
        </div>
      </div>
    </section>
  </main>
  <nav class="mobile-bottom-nav">
    <a :class="navClass('/home')" href="/home" @click.prevent="go('/home')"><i class="bi bi-house"></i><span>Home</span></a>
    <a :class="navClass('/meals')" href="/meals" @click.prevent="go('/meals')"><i class="bi bi-clock-history"></i><span>History</span></a>
    <a :class="navClass('/reports')" href="/reports" @click.prevent="go('/reports')"><i class="bi bi-file-earmark-pdf"></i><span>Reports</span></a>
    <a :class="navClass('/coach')" href="/coach" @click.prevent="go('/coach')"><i class="bi bi-robot"></i><span>Coach</span></a>
    <a :class="navClass('/profile')" href="/profile" @click.prevent="go('/profile')"><i class="bi bi-person"></i><span>Profile</span></a>
  </nav>
  <div v-if="toast" class="toast-line">{{ toast }}</div>
</div>
</template>
