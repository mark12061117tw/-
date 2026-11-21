#include <iostream>
#include <fstream>
#include <sstream>
#include <vector>
#include <string>
#include <set>
#include <map>
#include <algorithm>
#include <cctype>
#include <iomanip>
#include <cmath>

using namespace std;

// ---------------- 7 類旅遊興趣與微調設定列舉 ----------------
enum class InterestCategory {
    ArtCulture = 1,      // 1. 藝文展覽與建築美學
    FoodNightmarket,     // 2. 在地美食與夜市小吃
    CultureCreative,     // 3. 文創聚落與手作體驗
    NatureScenery,       // 4. 山林自然與海景夕陽
    ShoppingLeisure,     // 5. 百貨購物與商圈逛街
    ActiveOutdoor,       // 6. 單車騎行與動態體驗
    RelaxHealing         // 7. 慢活放空與溫泉療癒
};

enum class PaceMode {
    Relaxed,   // 悠閒（停留 x1.25）
    Normal,    // 普通（停留 x1.0）
    Compact    // 緊湊（停留 x0.75）
};

enum class RoutePreference {
    Fastest,         // 最快
    FewerTransfers,  // 較少轉乘
    LessWalking      // 較少步行
};

struct InterestConfig {
    bool enabled = false; // 是否勾選
    int intensity = 2;    // 興趣強度 1 ~ 3，預設 2
};

struct InterestInput {
    bool hasAnySelected = false; // 若為 false，代表不填 -> 所有類別一樣重要
    map<InterestCategory, InterestConfig> items;

    InterestInput() {
        for (int i = 1; i <= 7; ++i) {
            items[static_cast<InterestCategory>(i)] = InterestConfig{false, 2};
        }
    }

    int getWeight(InterestCategory cat) const {
        if (!hasAnySelected) return 2;
        const auto& cfg = items.at(cat);
        return cfg.enabled ? cfg.intensity : 0;
    }
};

struct CommonPlace {
    int id;
    string name;
    string area;
};

struct Spot {
    int id;
    string name;
    vector<InterestCategory> categories;
    set<string> mbtiTags;      // 適合的 MBTI 人格
    int openMinute;            // 每日開門時間（從 00:00 起算的分鐘數）
    int closeMinute;           // 每日關門時間
    set<int> closedDays;       // 公休日：0=日, 1=一, 2=二, 3=三, 4=四, 5=五, 6=六
    int baseStayMinutes;       // 基準建議停留時間（分鐘）
    double rating;             // 評分 1.0 ~ 5.0
    bool canMeal;              // 是否可作為用餐地點
};

// 先宣告快照結構，供 PlannerState 使用
struct PlannerSnapshot;

struct PlannerState {
    // 16 人格設定
    string primaryMbti;
    string secondaryMbti;

    // 必填：起點、日期、出發/結束時間
    string originName;
    string tripDate = "2026-11-22";
    int dayOfWeek = 0;           // 0=日 .. 6=六
    bool isWeekend = true;
    int availableStartMin = 540; // 09:00
    int availableEndMin = 1260;  // 21:00

    // 選填輸入
    InterestInput interests;
    string destinationName;      // 空字串代表「結束在最後一站」
    bool hasBudgetLimit = false; // false 代表「不限」
    int budgetTwd = 0;

    bool allowMrt = true;        // 捷運
    bool allowBus = true;        // 公車
    bool allowTrain = true;      // 台鐵
    bool allowYouBike = true;    // YouBike（步行一律允許）

    // 微調設定
    PaceMode pace = PaceMode::Normal;
    bool enableLunch = true;
    int lunchStartMin = 690;     // 11:30
    int lunchEndMin = 810;       // 13:30
    bool enableDinner = true;
    int dinnerStartMin = 1050;   // 17:30
    int dinnerEndMin = 1170;     // 19:30
    RoutePreference routePref = RoutePreference::Fastest;

    // 清單與多輪狀態管理
    int currentRound = 1;
    int roundAddedCount = 0;
    vector<int> mustGoIds;         // 清單 1：一定要去
    vector<int> optionalIds;       // 清單 2：可選/推薦清單
    vector<int> deletedIds;        // 已刪除（不想去/已移除）清單
};

// 完整狀態快照（解決 reinput 後 undo 無法還原設定的 Bug）
struct FullSnapshot {
    PlannerState savedState;
    string note;
};

// ---------------- 安全字串、數字與時間工具 ----------------
string trim(const string& s) {
    size_t a = s.find_first_not_of(" \t\r\n");
    if (a == string::npos) return "";
    size_t b = s.find_last_not_of(" \t\r\n");
    return s.substr(a, b - a + 1);
}

// [修復 Bug 1] 安全檢查全數字，避免 UTF-8 負值 char 觸發 ::isdigit 的 UB
bool isAllDigits(const string& s) {
    if (s.empty()) return false;
    return all_of(s.begin(), s.end(), [](unsigned char c) {
        return std::isdigit(c) != 0;
    });
}

// [修復 Bug 3] 嚴格整數轉換（防止溢位崩潰與 "123abc" 尾隨字元誤判）
bool tryParseInt(const string& s, int& outVal) {
    string t = trim(s);
    if (t.empty()) return false;
    try {
        size_t idx = 0;
        int val = stoi(t, &idx);
        if (idx != t.size()) return false;
        outVal = val;
        return true;
    } catch (...) {
        return false;
    }
}

string toUpper(string s) {
    transform(s.begin(), s.end(), s.begin(), [](unsigned char c) {
        return static_cast<char>(toupper(c));
    });
    return s;
}

string toLower(string s) {
    transform(s.begin(), s.end(), s.begin(), [](unsigned char c) {
        return static_cast<char>(tolower(c));
    });
    return s;
}

bool isValidMbti(const string& code) {
    static const set<string> valid16 = {
        "INTJ", "INTP", "ENTJ", "ENTP",
        "INFJ", "INFP", "ENFJ", "ENFP",
        "ISTJ", "ISFJ", "ESTJ", "ESFJ",
        "ISTP", "ISFP", "ESTP", "ESFP"
    };
    return valid16.count(code) > 0;
}

int parseTimeToMinutes(const string& hhmm) {
    string t = trim(hhmm);
    size_t colonPos = t.find(':');
    if (colonPos == string::npos) return -1;
    int h = -1, m = -1;
    if (!tryParseInt(t.substr(0, colonPos), h) || !tryParseInt(t.substr(colonPos + 1), m)) {
        return -1;
    }
    if (h < 0 || h > 29 || m < 0 || m >= 60) return -1;
    return h * 60 + m;
}

bool parseTimeRange(const string& rangeStr, int& outStart, int& outEnd) {
    size_t dashPos = rangeStr.find('-');
    if (dashPos == string::npos) return false;
    int s = parseTimeToMinutes(rangeStr.substr(0, dashPos));
    int e = parseTimeToMinutes(rangeStr.substr(dashPos + 1));
    if (s >= 0 && e > s) {
        outStart = s;
        outEnd = e;
        return true;
    }
    return false;
}

string formatMinutes(int mins) {
    int h = (mins / 60) % 24;
    int m = mins % 60;
    ostringstream oss;
    oss << setw(2) << setfill('0') << h << ":"
        << setw(2) << setfill('0') << m;
    return oss.str();
}

string formatDuration(int mins) {
    int h = mins / 60;
    int m = mins % 60;
    ostringstream oss;
    if (h > 0 && m > 0) oss << h << " 小時 " << m << " 分鐘";
    else if (h > 0) oss << h << " 小時";
    else oss << m << " 分鐘";
    return oss.str();
}

string dayName(int d) {
    static const char* names[] = {"週日", "週一", "週二", "週三", "週四", "週五", "週六"};
    return (d >= 0 && d <= 6) ? names[d] : "未知";
}

bool isLeapYear(int y) {
    return (y % 4 == 0 && y % 100 != 0) || (y % 400 == 0);
}

int daysInMonth(int y, int m) {
    static const int days[] = {0, 31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31};
    if (m == 2 && isLeapYear(y)) return 29;
    return (m >= 1 && m <= 12) ? days[m] : 0;
}

// [修復 Bug 3] 驗證真實月曆天數與尾隨多餘字元
bool parseDateInput(const string& raw, string& outDateStr, int& outDayOfWeek) {
    string s = trim(raw);
    if (s.empty()) return false;

    if (s.size() == 1 && isAllDigits(s)) {
        int d = s[0] - '0';
        if (d >= 0 && d <= 6) {
            outDayOfWeek = d;
            outDateStr = "指定" + string(dayName(d));
            return true;
        }
        return false;
    }

    int y = 0, m = 0, d = 0;
    char sep1 = 0, sep2 = 0;
    stringstream ss(s);
    if (!(ss >> y >> sep1 >> m >> sep2 >> d)) return false;
    string leftover;
    if (ss >> leftover) return false; // 拒絕 "2026-11-22abc"
    if ((sep1 != '-' && sep1 != '/') || (sep2 != '-' && sep2 != '/')) return false;
    if (y < 1900 || m < 1 || m > 12 || d < 1 || d > daysInMonth(y, m)) return false;

    static const int t[] = {0, 3, 2, 5, 0, 3, 5, 1, 4, 6, 2, 4};
    int yAdj = (m < 3) ? y - 1 : y;
    int dow = (yAdj + yAdj / 4 - yAdj / 100 + yAdj / 400 + t[m - 1] + d) % 7;

    ostringstream oss;
    oss << setw(4) << setfill('0') << y << "-"
        << setw(2) << setfill('0') << m << "-"
        << setw(2) << setfill('0') << d;
    outDateStr = oss.str();
    outDayOfWeek = dow;
    return true;
}

// ---------------- 興趣類別與微調標籤 ----------------
string categoryLabel(InterestCategory cat) {
    switch (cat) {
        case InterestCategory::ArtCulture:      return "藝文展覽與建築美學";
        case InterestCategory::FoodNightmarket: return "在地美食與夜市小吃";
        case InterestCategory::CultureCreative: return "文創聚落與手作體驗";
        case InterestCategory::NatureScenery:   return "山林自然與海景夕陽";
        case InterestCategory::ShoppingLeisure: return "百貨購物與商圈逛街";
        case InterestCategory::ActiveOutdoor:   return "單車騎行與動態體驗";
        case InterestCategory::RelaxHealing:    return "慢活放空與溫泉療癒";
    }
    return "";
}

double paceMultiplier(PaceMode pace) {
    switch (pace) {
        case PaceMode::Relaxed: return 1.25;
        case PaceMode::Normal:  return 1.00;
        case PaceMode::Compact: return 0.75;
    }
    return 1.00;
}

string paceLabel(PaceMode pace) {
    switch (pace) {
        case PaceMode::Relaxed: return "悠閒（停留 x1.25）";
        case PaceMode::Normal:  return "普通（停留 x1.0）";
        case PaceMode::Compact: return "緊湊（停留 x0.75）";
    }
    return "普通";
}

string routePrefLabel(RoutePreference pref) {
    switch (pref) {
        case RoutePreference::Fastest:        return "最快";
        case RoutePreference::FewerTransfers: return "較少轉乘";
        case RoutePreference::LessWalking:    return "較少步行";
    }
    return "最快";
}

vector<InterestCategory> defaultInterestsByMbti(const string& mbti) {
    if (mbti == "INTJ") return {InterestCategory::ArtCulture, InterestCategory::RelaxHealing};
    if (mbti == "INTP") return {InterestCategory::ArtCulture, InterestCategory::NatureScenery};
    if (mbti == "ENTJ") return {InterestCategory::ShoppingLeisure, InterestCategory::FoodNightmarket};
    if (mbti == "ENTP") return {InterestCategory::CultureCreative, InterestCategory::FoodNightmarket};
    if (mbti == "INFJ") return {InterestCategory::ArtCulture, InterestCategory::RelaxHealing};
    if (mbti == "INFP") return {InterestCategory::NatureScenery, InterestCategory::RelaxHealing};
    if (mbti == "ENFJ") return {InterestCategory::CultureCreative, InterestCategory::FoodNightmarket};
    if (mbti == "ENFP") return {InterestCategory::CultureCreative, InterestCategory::ActiveOutdoor};
    if (mbti == "ISTJ") return {InterestCategory::ArtCulture, InterestCategory::FoodNightmarket};
    if (mbti == "ISFJ") return {InterestCategory::RelaxHealing, InterestCategory::FoodNightmarket};
    if (mbti == "ESTJ") return {InterestCategory::NatureScenery, InterestCategory::ShoppingLeisure};
    if (mbti == "ESFJ") return {InterestCategory::FoodNightmarket, InterestCategory::ShoppingLeisure};
    if (mbti == "ISTP") return {InterestCategory::ActiveOutdoor, InterestCategory::NatureScenery};
    if (mbti == "ISFP") return {InterestCategory::NatureScenery, InterestCategory::CultureCreative};
    if (mbti == "ESTP") return {InterestCategory::ActiveOutdoor, InterestCategory::ShoppingLeisure};
    return {InterestCategory::FoodNightmarket, InterestCategory::ActiveOutdoor}; // ESFP
}

// ---------------- 常用起終點與預設景點資料 ----------------
vector<CommonPlace> buildCommonPlaces() {
    return {
        {1, "臺中車站", "中區"},
        {2, "高鐵臺中站", "烏日區"},
        {3, "市政府站（臺中市政府）", "西屯區"},
        {4, "逢甲商圈", "西屯區"},
        {5, "一中商圈", "北區"},
        {6, "勤美草悟道", "西區"}
    };
}

vector<Spot> buildDefaultSpots() {
    return {
        {1, "臺中國家歌劇院", {InterestCategory::ArtCulture, InterestCategory::RelaxHealing}, {"INTJ", "ENTP", "INFJ"}, 690, 1260, {1}, 120, 4.8, false},
        {2, "國立自然科學博物館", {InterestCategory::ArtCulture}, {"INTP", "INTJ", "ISTJ"}, 540, 1020, {1}, 150, 4.7, false},
        {3, "台中三井 LaLaport", {InterestCategory::ShoppingLeisure, InterestCategory::FoodNightmarket}, {"ENTJ", "ESFJ", "ESTP"}, 660, 1320, {}, 150, 4.5, true},
        {4, "富興工廠 1962", {InterestCategory::CultureCreative, InterestCategory::ArtCulture}, {"ENTP", "ENFP", "INFP"}, 660, 1200, {1}, 90, 4.6, false},
        {5, "國立臺灣美術館", {InterestCategory::ArtCulture, InterestCategory::RelaxHealing}, {"INFJ", "INFP", "INTJ"}, 540, 1020, {1}, 120, 4.7, false},
        {6, "薰衣草森林（新社店）", {InterestCategory::NatureScenery, InterestCategory::RelaxHealing}, {"INFP", "ISFJ", "ENFP"}, 630, 1110, {}, 180, 4.5, true},
        {7, "審計新村", {InterestCategory::CultureCreative, InterestCategory::ShoppingLeisure}, {"ENFJ", "ENFP", "INFP"}, 660, 1200, {}, 90, 4.4, false},
        {8, "彩虹眷村", {InterestCategory::CultureCreative, InterestCategory::NatureScenery}, {"ENFP", "ESFP", "ISFP"}, 540, 1080, {1}, 60, 4.1, false},
        {9, "台中州廳", {InterestCategory::ArtCulture}, {"ISTJ", "INFJ", "ESTJ"}, 540, 1020, {1}, 75, 4.6, false},
        {10, "谷關溫泉區", {InterestCategory::RelaxHealing, InterestCategory::NatureScenery}, {"ISFJ", "INFP", "ISTP"}, 480, 1260, {}, 180, 4.5, true},
        {11, "中央公園", {InterestCategory::NatureScenery, InterestCategory::RelaxHealing}, {"ESTJ", "ISTP", "ENFJ"}, 360, 1380, {}, 90, 4.5, false},
        {12, "第二市場", {InterestCategory::FoodNightmarket}, {"ESFJ", "ISTJ", "ESTP"}, 480, 900, {}, 90, 4.5, true},
        {13, "后豐鐵馬道", {InterestCategory::ActiveOutdoor, InterestCategory::NatureScenery}, {"ISTP", "ESTP", "ENFP"}, 480, 1140, {}, 120, 4.7, false},
        {14, "高美濕地", {InterestCategory::NatureScenery, InterestCategory::RelaxHealing}, {"ISFP", "INFP", "ESFP"}, 540, 1140, {}, 120, 4.7, false},
        {15, "麗寶 Outlet Mall", {InterestCategory::ShoppingLeisure, InterestCategory::ActiveOutdoor}, {"ESTP", "ENTJ", "ESFJ"}, 660, 1290, {}, 150, 4.5, true},
        {16, "逢甲夜市", {InterestCategory::FoodNightmarket, InterestCategory::ShoppingLeisure}, {"ESFP", "ESTP", "ENFP"}, 960, 1380, {}, 120, 4.4, true}
    };
}

int getAdjustedStayMinutes(const Spot& s, PaceMode pace) {
    return max(15, static_cast<int>(round(s.baseStayMinutes * paceMultiplier(pace))));
}

// ---------------- 推薦分數與「先擋」衝突檢查 ----------------
double calcRecommendationScore(const Spot& s, const PlannerState& st) {
    double score = s.rating * 10.0;
    if (s.mbtiTags.count(st.primaryMbti)) score += 25.0;
    if (!st.secondaryMbti.empty() && s.mbtiTags.count(st.secondaryMbti)) score += 15.0;

    int interestBonus = 0;
    for (InterestCategory cat : s.categories) {
        interestBonus += st.interests.getWeight(cat) * 12;
    }
    score += interestBonus;
    return score;
}

string checkPreBlockReason(const Spot& s, const PlannerState& st) {
    if (s.closedDays.count(st.dayOfWeek)) {
        return "當天公休（" + dayName(st.dayOfWeek) + "）";
    }
    if (st.availableEndMin <= s.openMinute || st.availableStartMin >= s.closeMinute) {
        return "可玩時段與營業時間不重疊（營業 " + formatMinutes(s.openMinute) + "-" + formatMinutes(s.closeMinute) + "）";
    }
    int stay = getAdjustedStayMinutes(s, st.pace);
    if (stay > (st.availableEndMin - st.availableStartMin)) {
        return "單點調整後停留（" + to_string(stay) + "分）超過全日可用時間";
    }
    return "";
}

const Spot* findSpotById(const vector<Spot>& spots, int id) {
    for (const auto& s : spots) {
        if (s.id == id) return &s;
    }
    return nullptr;
}

int findSpotIdByName(const vector<Spot>& spots, const string& query) {
    string q = trim(query);
    if (q.empty()) return -1;
    for (const auto& s : spots) {
        if (s.name == q) return s.id;
    }
    for (const auto& s : spots) {
        if (s.name.find(q) != string::npos) return s.id;
    }
    return -1;
}

int resolveSpotId(const vector<Spot>& spots, const string& arg) {
    int id = -1;
    if (isAllDigits(arg)) {
        if (tryParseInt(arg, id)) return id;
        return -1;
    }
    return findSpotIdByName(spots, arg);
}

int calcTotalMustGoMinutes(const vector<Spot>& spots, const PlannerState& st) {
    int total = 0;
    for (int id : st.mustGoIds) {
        const Spot* s = findSpotById(spots, id);
        if (s) total += getAdjustedStayMinutes(*s, st.pace);
    }
    return total;
}

// ---------------- 輸入與微調設定流程 ----------------
string promptPlace(const vector<CommonPlace>& places, const string& title, bool required, const string& emptyMeaning) {
    while (true) {
        cout << "\n[" << title << "] 常用地點：\n";
        for (const auto& p : places) {
            cout << "  " << p.id << ". " << p.name << "（" << p.area << "）\n";
        }
        if (!required) {
            cout << "請輸入編號、地點名稱，或直接按 Enter（" << emptyMeaning << "）：";
        } else {
            cout << "請輸入編號或地點名稱（必填）：";
        }

        string line;
        getline(cin, line);
        line = trim(line);

        if (line.empty()) {
            if (!required) return "";
            cout << "此欄位為必填，請輸入編號或地點名稱。\n";
            continue;
        }

        if (isAllDigits(line)) {
            int idx = -1;
            if (tryParseInt(line, idx)) {
                for (const auto& p : places) {
                    if (p.id == idx) return p.name;
                }
            }
            cout << "查無編號 " << line << "，請重新輸入。\n";
            continue;
        }

        return line;
    }
}

InterestInput promptInterestInput(const string& mbti) {
    InterestInput input;
    auto suggested = defaultInterestsByMbti(mbti);

    cout << "\n--- 旅遊興趣（7 類勾選，直接按 Enter 表示不填＝所有類別同等重要） ---\n";
    for (int i = 1; i <= 7; ++i) {
        auto cat = static_cast<InterestCategory>(i);
        cout << "  " << i << ". " << categoryLabel(cat) << "\n";
    }
    cout << "（提示：" << mbti << " 常見偏好為 ";
    for (size_t i = 0; i < suggested.size(); ++i) {
        cout << static_cast<int>(suggested[i]) << "." << categoryLabel(suggested[i]);
        if (i + 1 < suggested.size()) cout << "、";
    }
    cout << "）\n";
    cout << "請輸入想勾選的編號（以空白或逗號分隔，例如 1 2 或 2,4,7）：";

    string line;
    getline(cin, line);
    for (char& ch : line) {
        if (ch == ',') ch = ' ';
    }

    stringstream ss(line);
    string token;
    while (ss >> token) {
        int idx = -1;
        if (tryParseInt(token, idx) && idx >= 1 && idx <= 7) {
            auto cat = static_cast<InterestCategory>(idx);
            input.items[cat].enabled = true;
            input.hasAnySelected = true;
        }
    }

    if (!input.hasAnySelected) {
        cout << "[預設生效] 未勾選興趣，所有 7 類均以相同權重參與推薦。\n";
        return input;
    }

    cout << "請為已勾選類別設定興趣強度（1～3，直接按 Enter 預設為 2）：\n";
    for (int i = 1; i <= 7; ++i) {
        auto cat = static_cast<InterestCategory>(i);
        if (!input.items[cat].enabled) continue;
        cout << "  [" << categoryLabel(cat) << "] 強度 (1-3, 預設 2)：";
        string valStr;
        getline(cin, valStr);
        valStr = trim(valStr);
        if (!valStr.empty()) {
            int v = -1;
            if (tryParseInt(valStr, v) && v >= 1 && v <= 3) {
                input.items[cat].intensity = v;
            } else {
                cout << "  強度需在 1～3 之間，已套用預設 2。\n";
            }
        }
    }

    return input;
}

void promptFullInputAndTuning(PlannerState& st, const vector<CommonPlace>& places) {
    cout << "\n================ 1. 16 人格設定 ================\n";
    while (true) {
        cout << "請輸入主要 MBTI 人格（16 型之一，例如 INTJ、INFP、ESFP）：";
        string code;
        getline(cin, code);
        code = toUpper(trim(code));
        if (isValidMbti(code)) {
            st.primaryMbti = code;
            break;
        }
        cout << "輸入無效，請輸入正確的四字母 MBTI。\n";
    }

    cout << "若為雙人同行，請輸入第二位 MBTI 人格（單人請直接按 Enter）：";
    string sec;
    getline(cin, sec);
    sec = toUpper(trim(sec));
    if (sec.empty()) {
        st.secondaryMbti.clear();
    } else if (isValidMbti(sec)) {
        st.secondaryMbti = sec;
    } else {
        cout << "第二人格格式不符，已略過，維持單人模式。\n";
        st.secondaryMbti.clear();
    }

    cout << "\n================ 2. 必填旅遊資訊 ================\n";
    st.originName = promptPlace(places, "起點", true, "");

    while (true) {
        cout << "請輸入旅遊日期（YYYY-MM-DD，或直接輸入星期 0=日..6=六，例如 2026-11-22 或 6）：";
        string dInput;
        getline(cin, dInput);
        if (parseDateInput(dInput, st.tripDate, st.dayOfWeek)) {
            st.isWeekend = (st.dayOfWeek == 0 || st.dayOfWeek == 6);
            break;
        }
        cout << "日期格式無效，請輸入有效日期如 2026-11-22 或 0~6。\n";
    }

    while (true) {
        cout << "請輸入出發時間與結束時間（格式 HH:MM HH:MM，例如 09:00 21:00）：";
        string tInput;
        getline(cin, tInput);
        stringstream ss(trim(tInput));
        string startS, endS, extra;
        if ((ss >> startS >> endS) && !(ss >> extra)) {
            int sMin = parseTimeToMinutes(startS);
            int eMin = parseTimeToMinutes(endS);
            if (sMin >= 0 && eMin > sMin) {
                st.availableStartMin = sMin;
                st.availableEndMin = eMin;
                break;
            }
        }
        cout << "時間格式錯誤，請輸入如 09:00 21:00（結束需晚於出發）。\n";
    }

    cout << "\n================ 3. 選填條件設定 ================\n";
    st.interests = promptInterestInput(st.primaryMbti);

    st.destinationName = promptPlace(places, "終點", false, "結束在最後一站");

    cout << "\n請輸入預算上限（新台幣元，直接按 Enter 表示「不限」）：";
    string budgetStr;
    getline(cin, budgetStr);
    budgetStr = trim(budgetStr);
    if (budgetStr.empty()) {
        st.hasBudgetLimit = false;
        st.budgetTwd = 0;
    } else {
        int b = -1;
        if (tryParseInt(budgetStr, b) && b >= 0) {
            st.hasBudgetLimit = true;
            st.budgetTwd = b;
        } else {
            cout << "預算格式錯誤或為負數，已設為「不限」。\n";
            st.hasBudgetLimit = false;
            st.budgetTwd = 0;
        }
    }

    cout << "\n可以搭的交通工具（捷運、公車、台鐵、YouBike；直接按 Enter＝全部可以，步行一律允許）：\n"
         << "請輸入要保留的編號（1=捷運, 2=公車, 3=台鐵, 4=YouBike，以空白或逗號分隔）：";
    string transLine;
    getline(cin, transLine);
    transLine = trim(transLine);
    if (transLine.empty()) {
        st.allowMrt = st.allowBus = st.allowTrain = st.allowYouBike = true;
    } else {
        bool m = false, b = false, tr = false, yb = false;
        for (char& ch : transLine) {
            if (ch == ',') ch = ' ';
        }
        stringstream tss(transLine);
        string tok;
        while (tss >> tok) {
            int tid = -1;
            if (tryParseInt(tok, tid)) {
                if (tid == 1) m = true;
                else if (tid == 2) b = true;
                else if (tid == 3) tr = true;
                else if (tid == 4) yb = true;
            }
        }
        st.allowMrt = m;
        st.allowBus = b;
        st.allowTrain = tr;
        st.allowYouBike = yb;
    }

    cout << "\n================ 4. 微調設定 ================\n";
    cout << "步調（1=悠閒 x1.25, 2=普通 x1.0 [預設], 3=緊湊 x0.75），直接按 Enter 預設普通：";
    string paceStr;
    getline(cin, paceStr);
    paceStr = trim(paceStr);
    if (paceStr == "1") st.pace = PaceMode::Relaxed;
    else if (paceStr == "3") st.pace = PaceMode::Compact;
    else st.pace = PaceMode::Normal;

    // [修復 Bug 4] 真正實作用餐時段自訂解析
    cout << "用餐時段設定（直接按 Enter 套用預設：午餐 11:30-13:30、晚餐 17:30-19:30；輸入 off 可關閉；或輸入 12:00-13:30 18:00-19:30 自訂）：";
    string mealStr;
    getline(cin, mealStr);
    mealStr = trim(mealStr);
    st.enableLunch = true;
    st.lunchStartMin = 690;
    st.lunchEndMin = 810;
    st.enableDinner = true;
    st.dinnerStartMin = 1050;
    st.dinnerEndMin = 1170;

    if (toLower(mealStr) == "off") {
        st.enableLunch = false;
        st.enableDinner = false;
    } else if (!mealStr.empty()) {
        stringstream mss(mealStr);
        string r1, r2;
        mss >> r1 >> r2;
        int ls, le, ds, de;
        if (!r1.empty() && !r2.empty() && parseTimeRange(r1, ls, le) && parseTimeRange(r2, ds, de)) {
            st.lunchStartMin = ls;
            st.lunchEndMin = le;
            st.dinnerStartMin = ds;
            st.dinnerEndMin = de;
            cout << "已套用自訂用餐時段：午餐 " << formatMinutes(ls) << "-" << formatMinutes(le)
                 << "、晚餐 " << formatMinutes(ds) << "-" << formatMinutes(de) << "。\n";
        } else {
            cout << "格式無法辨識，已保留預設午餐 11:30-13:30 與晚餐 17:30-19:30。\n";
        }
    }

    cout << "路線偏好（1=最快 [預設], 2=較少轉乘, 3=較少步行），直接按 Enter 預設最快：";
    string prefStr;
    getline(cin, prefStr);
    prefStr = trim(prefStr);
    if (prefStr == "2") st.routePref = RoutePreference::FewerTransfers;
    else if (prefStr == "3") st.routePref = RoutePreference::LessWalking;
    else st.routePref = RoutePreference::Fastest;
}

// ---------------- 快照與清單顯示 ----------------
void saveSnapshot(vector<FullSnapshot>& history, const PlannerState& st, const string& note) {
    history.push_back(FullSnapshot{st, note});
}

bool undoLastSnapshot(vector<FullSnapshot>& history, PlannerState& st) {
    if (history.empty()) return false;
    st = history.back().savedState;
    history.pop_back();
    return true;
}

// [修復 Bug 2] reinput 後自動剔除與新條件衝突的已選景點
void revalidateMustGoList(const vector<Spot>& spots, PlannerState& st) {
    vector<int> validIds;
    for (int id : st.mustGoIds) {
        const Spot* s = findSpotById(spots, id);
        if (!s) continue;
        string reason = checkPreBlockReason(*s, st);
        if (!reason.empty()) {
            cout << "[自動剔除] 「" << s->name << "」因新條件衝突已移出一定要去清單（" << reason << "）。\n";
        } else {
            validIds.push_back(id);
        }
    }
    st.mustGoIds = validIds;
}

void refreshOptionalList(const vector<Spot>& spots, PlannerState& st) {
    set<int> excluded(st.mustGoIds.begin(), st.mustGoIds.end());
    excluded.insert(st.deletedIds.begin(), st.deletedIds.end());

    vector<pair<double, int>> scored;
    for (const auto& s : spots) {
        if (excluded.count(s.id)) continue;
        scored.push_back({calcRecommendationScore(s, st), s.id});
    }
    sort(scored.begin(), scored.end(), [](const auto& a, const auto& b) {
        if (a.first != b.first) return a.first > b.first;
        return a.second < b.second;
    });

    st.optionalIds.clear();
    for (const auto& item : scored) {
        st.optionalIds.push_back(item.second);
    }
}

void printSummaryHeader(const vector<Spot>& spots, const PlannerState& st) {
    int selectedMin = calcTotalMustGoMinutes(spots, st);
    int availableMin = max(0, st.availableEndMin - st.availableStartMin);

    cout << "\n==================== 第 " << st.currentRound << " 輪輸入狀態 ====================\n";
    cout << "人格：" << st.primaryMbti
         << (st.secondaryMbti.empty() ? "" : (" + " + st.secondaryMbti))
         << " | 日期：" << st.tripDate << "（" << dayName(st.dayOfWeek) << (st.isWeekend ? "・假日" : "・平日") << "）\n";
    cout << "起點：" << st.originName
         << " | 終點：" << (st.destinationName.empty() ? "結束在最後一站" : st.destinationName)
         << " | 時段：" << formatMinutes(st.availableStartMin) << " - " << formatMinutes(st.availableEndMin) << "\n";
    cout << "預算：" << (st.hasBudgetLimit ? ("NT$" + to_string(st.budgetTwd)) : "不限")
         << " | 交通：";
    vector<string> trans;
    if (st.allowMrt) trans.push_back("捷運");
    if (st.allowBus) trans.push_back("公車");
    if (st.allowTrain) trans.push_back("台鐵");
    if (st.allowYouBike) trans.push_back("YouBike");
    trans.push_back("步行");
    for (size_t i = 0; i < trans.size(); ++i) {
        cout << trans[i] << (i + 1 < trans.size() ? "／" : "");
    }
    cout << "\n微調：步調=" << paceLabel(st.pace)
         << " | 路線=" << routePrefLabel(st.routePref)
         << " | 用餐=";
    if (!st.enableLunch && !st.enableDinner) {
        cout << "關閉\n";
    } else {
        cout << "午 " << formatMinutes(st.lunchStartMin) << "-" << formatMinutes(st.lunchEndMin)
             << " / 晚 " << formatMinutes(st.dinnerStartMin) << "-" << formatMinutes(st.dinnerEndMin) << "\n";
    }

    cout << "[時間條] 已選停留約 " << formatDuration(selectedMin)
         << " / 全日可用 " << formatDuration(availableMin)
         << "（本輪新指定 " << st.roundAddedCount << " / 上限 8 個）\n";

    cout << "\n【清單 1：一定要去（" << st.mustGoIds.size() << " 個）】\n";
    if (st.mustGoIds.empty()) {
        cout << "  （目前空白）\n";
    } else {
        for (size_t i = 0; i < st.mustGoIds.size(); ++i) {
            const Spot* s = findSpotById(spots, st.mustGoIds[i]);
            if (s) {
                cout << "  " << (i + 1) << ". [" << s->id << "] " << s->name
                     << "（調整停留 " << getAdjustedStayMinutes(*s, st.pace) << " 分）\n";
            }
        }
    }

    cout << "\n【清單 2：推薦 / 可選景點（依人格與 7 類興趣權重排序）】\n";
    for (int id : st.optionalIds) {
        const Spot* s = findSpotById(spots, id);
        if (!s) continue;
        string blockReason = checkPreBlockReason(*s, st);
        double score = calcRecommendationScore(*s, st);
        if (!blockReason.empty()) {
            cout << "  [灰色不可選] ID:" << s->id << " " << s->name
                 << " -> " << blockReason << "\n";
        } else {
            cout << "  [可選] ID:" << s->id << " " << s->name
                 << " | 推薦分:" << fixed << setprecision(1) << score
                 << " | 停留:" << getAdjustedStayMinutes(*s, st.pace) << "分"
                 << (s->canMeal ? " | 可用餐" : "") << "\n";
        }
    }

    cout << "\n【已刪除 / 不想去清單（" << st.deletedIds.size() << " 個）】";
    if (st.deletedIds.empty()) {
        cout << " 無\n";
    } else {
        cout << "\n";
        for (int id : st.deletedIds) {
            const Spot* s = findSpotById(spots, id);
            if (s) cout << "  - ID:" << s->id << " " << s->name << "\n";
        }
    }
}

int main() {
#ifdef _WIN32
    system("chcp 65001 > nul");
#endif

    vector<CommonPlace> commonPlaces = buildCommonPlaces();
    vector<Spot> spots = buildDefaultSpots();

    PlannerState state;
    vector<FullSnapshot> history;

    promptFullInputAndTuning(state, commonPlaces);
    refreshOptionalList(spots, state);

    while (true) {
        printSummaryHeader(spots, state);

        cout << "\n操作指令：\n"
             << "  add <景點ID或名稱>     加入「一定要去」（單輪新指定上限 8 個，會先擋公休/時段不合）\n"
             << "  remove <景點ID或名稱>  從「一定要去」或「可選清單」移出並放入「已刪除清單」\n"
             << "  restore <景點ID或名稱> 從「已刪除清單」還原回可選清單\n"
             << "  next                   進入下一輪（重算本輪新增上限 8 個並存檔快照）\n"
             << "  undo                   返回上一輪/上一步快照\n"
             << "  reinput                重新填寫輸入與微調設定\n"
             << "  done                   確認輸入並輸出最終整理結果\n"
             << "請輸入指令：";

        string line;
        if (!getline(cin, line)) break;
        line = trim(line);
        if (line.empty()) continue;

        stringstream ss(line);
        string cmd;
        ss >> cmd;
        string arg;
        getline(ss, arg);
        arg = trim(arg);
        string lowerCmd = toLower(cmd);

        if (lowerCmd == "done" || lowerCmd == "exit") {
            break;
        } else if (lowerCmd == "reinput") {
            saveSnapshot(history, state, "重新填寫設定前快照");
            promptFullInputAndTuning(state, commonPlaces);
            revalidateMustGoList(spots, state);
            refreshOptionalList(spots, state);
        } else if (lowerCmd == "undo") {
            if (undoLastSnapshot(history, state)) {
                cout << "[已還原] 回到第 " << state.currentRound << " 輪狀態。\n";
            } else {
                cout << "[提示] 沒有可返回的上一步快照。\n";
            }
        } else if (lowerCmd == "next") {
            saveSnapshot(history, state, "進入下一輪前快照");
            state.currentRound++;
            state.roundAddedCount = 0;
            refreshOptionalList(spots, state);
            cout << "[新回合] 已進入第 " << state.currentRound << " 輪，本輪新指定上限為 8 個。\n";
        } else if (lowerCmd == "add") {
            if (state.roundAddedCount >= 8) {
                cout << "[衝突攔截] 本輪新指定的「一定要去」已達上限 8 個！請輸入 next 進入下一輪或先移除部分景點。\n";
                continue;
            }
            int targetId = resolveSpotId(spots, arg);
            const Spot* s = findSpotById(spots, targetId);
            if (!s) {
                cout << "[找不到地點] 查無「" << arg << "」，請輸入正確的景點 ID 或名稱。\n";
                continue;
            }

            string blockReason = checkPreBlockReason(*s, state);
            if (!blockReason.empty()) {
                cout << "[先擋攔截] 無法加入「" << s->name << "」：" << blockReason << "。\n";
                continue;
            }

            if (find(state.mustGoIds.begin(), state.mustGoIds.end(), s->id) != state.mustGoIds.end()) {
                cout << "[提示] 「" << s->name << "」已在「一定要去」清單中。\n";
                continue;
            }

            saveSnapshot(history, state, "加入景點：" + s->name);
            state.mustGoIds.push_back(s->id);
            state.deletedIds.erase(remove(state.deletedIds.begin(), state.deletedIds.end(), s->id), state.deletedIds.end());
            state.roundAddedCount++;
            refreshOptionalList(spots, state);
            cout << "[已加入] 「" << s->name << "」已加入一定要去清單。\n";
        } else if (lowerCmd == "remove") {
            int targetId = resolveSpotId(spots, arg);
            const Spot* s = findSpotById(spots, targetId);
            if (!s) {
                cout << "[找不到地點] 查無「" << arg << "」。\n";
                continue;
            }

            auto mustIt = find(state.mustGoIds.begin(), state.mustGoIds.end(), targetId);
            auto optIt = find(state.optionalIds.begin(), state.optionalIds.end(), targetId);

            if (mustIt == state.mustGoIds.end() && optIt == state.optionalIds.end()) {
                cout << "[提示] 「" << s->name << "」不在可移除的清單中（可能已在已刪除清單）。\n";
                continue;
            }

            saveSnapshot(history, state, "移除景點：" + s->name);
            if (mustIt != state.mustGoIds.end()) {
                state.mustGoIds.erase(mustIt);
                if (state.roundAddedCount > 0) state.roundAddedCount--;
            }
            if (find(state.deletedIds.begin(), state.deletedIds.end(), targetId) == state.deletedIds.end()) {
                state.deletedIds.push_back(targetId);
            }
            refreshOptionalList(spots, state);
            cout << "[已移除] 已將「" << s->name << "」移至已刪除清單。\n";
        } else if (lowerCmd == "restore") {
            int targetId = resolveSpotId(spots, arg);
            auto it = find(state.deletedIds.begin(), state.deletedIds.end(), targetId);
            if (it == state.deletedIds.end()) {
                cout << "[提示] 「已刪除清單」中找不到「" << arg << "」。\n";
                continue;
            }

            const Spot* s = findSpotById(spots, *it);
            saveSnapshot(history, state, "還原景點：" + (s ? s->name : to_string(*it)));
            state.deletedIds.erase(it);
            refreshOptionalList(spots, state);
            cout << "[已還原] 已將「" << (s ? s->name : "") << "」還原至可選清單。\n";
        } else {
            cout << "[未知指令] 請輸入 add / remove / restore / next / undo / reinput / done。\n";
        }
    }

    cout << "\n==================== 最終輸入設定摘要 ====================\n";
    cout << "MBTI 人格：" << state.primaryMbti
         << (state.secondaryMbti.empty() ? "" : (" + " + state.secondaryMbti)) << "\n";
    cout << "日期與時段：" << state.tripDate << "（" << dayName(state.dayOfWeek) << "） "
         << formatMinutes(state.availableStartMin) << " - " << formatMinutes(state.availableEndMin) << "\n";
    cout << "起點：" << state.originName
         << " | 終點：" << (state.destinationName.empty() ? "結束在最後一站" : state.destinationName) << "\n";
    cout << "預算：" << (state.hasBudgetLimit ? ("NT$" + to_string(state.budgetTwd)) : "不限") << "\n";
    cout << "微調：步調=" << paceLabel(state.pace)
         << "、路線=" << routePrefLabel(state.routePref) << "\n";

    cout << "\n「一定要去」清單（共 " << state.mustGoIds.size() << " 個）：\n";
    int totalStay = 0;
    for (size_t i = 0; i < state.mustGoIds.size(); ++i) {
        const Spot* s = findSpotById(spots, state.mustGoIds[i]);
        if (s) {
            int stay = getAdjustedStayMinutes(*s, state.pace);
            cout << "  " << (i + 1) << ". " << s->name << "（停留 " << stay << " 分鐘）\n";
            totalStay += stay;
        }
    }
    cout << "總建議停留時間：" << formatDuration(totalStay) << "\n";
    return 0;
}
