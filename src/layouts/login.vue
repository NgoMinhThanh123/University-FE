<template>
  <div class="content">
    <div class="container">
      <div class="row">
        <div class="col-md-6">
          <img
            src="../assets/undraw_remotely_2j6y.svg"
            alt="Image"
            class="img-fluid"
          />
        </div>
        <div class="col-md-6 contents">
          <div class="row justify-content-center">
            <div class="col-md-8">
              <div class="mb-4">
                <h3>Đăng nhập</h3>
                <div v-if="errorMessage" class="alert alert-danger">
                  {{ errorMessage }}
                </div>
              </div>
              <form action="#" method="post" @submit.prevent="login">
                <div class="form-group first">
                  <label for="username">Username</label>
                  <input
                    type="text"
                    class="form-control"
                    id="username"
                    required
                    v-model="user.username"
                  />
                </div>
                <div class="form-group last mb-4">
                  <label for="password">Password</label>
                  <input
                    type="password"
                    class="form-control"
                    id="password"
                    required
                    v-model="user.password"
                  />
                </div>

                <input
                  type="submit"
                  value="Đăng nhập"
                  class="btn btn-block btn-primary"
                />
                <div
                  class="d-flex mt-3 align-items-center"
                  style="justify-content: center"
                >
                  <span class="registerHover" style="margin-right: 10px">
                    <router-link to="/" class="nav-link">
                      Trang chủ
                    </router-link>
                  </span>
                  <span class="registerHover">
                    <router-link to="/register" class="nav-link">
                      Đăng ký
                    </router-link>
                  </span>
                </div>
              </form>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>


<script>
import Apis, { endpoints, authApi } from "../configs/Apis.js";
import VueCookies from "vue-cookies";
import { mapState } from "vuex";
import { getAuth, signInWithEmailAndPassword } from "firebase/auth";
import { getFirestore, query, where, getDocs, collection } from "firebase/firestore";
import firebase from "../service/firebase";

export default {
  name: "Login",
  computed: {
    ...mapState(["store"]),
  },
  data() {
    return {
      user: {
        username: "",
        password: "",
      },
      errorMessage: "",
    };
  },

  methods: {
    async getUserByUsername(username) {
      try {
        const res = await authApi().get(
          endpoints["get-user"].replace("{username}", username)
        );
        return res.data;
      } catch (error) {
        console.log(error);
      }
    },
    async login() {
      try {
        const res = await Apis.post(`${endpoints["login"]}`, {
          username: this.user.username,
          password: this.user.password,
        });
        console.log(res.status);
        if (res.status === 400) {
          this.errorMessage = "Tài khoản hoặc mật khẩu của bạn không đúng!!";
        } else {
          console.log(res.data);
          this.errorMessage = "";
          VueCookies.set("token", res.data.accessToken);

          let { data } = await authApi().get(endpoints["current-user"]);
          VueCookies.set("user", data);

          await this.$store.dispatch("login", data);

          try {
            const u = await this.getUserByUsername(this.user.username);
            console.log("get-user", u);
            if (u) {
              const studentUsername = u.username;
            //   const studentInfo = await authApi().get(
            //     endpoints["get-student-by-username"].replace(
            //       "{username}",
            //       studentUsername
            //     )
            //   );

              const userEmail = u.email;

              const auth = getAuth(firebase); // Sử dụng auth từ Firebase Modular SDK
              const res = await signInWithEmailAndPassword(
                auth,
                userEmail,
                this.user.password
              );
              console.log("res.user",res.user);
              const user = res.user;

              if (user) {
                // Kiểm tra nếu người dùng đã đăng nhập thành công
                const db = getFirestore(firebase); // Sử dụng db từ Firebase Modular SDK
                const q = query(
                  collection(db, "users"),
                  where("id", "==", user.uid)
                );
                console.log("user.uid",user.uid);

                const querySnapshot = await getDocs(q);

                console.log("querySnapshot",querySnapshot);

                if (!querySnapshot.empty) {
                  querySnapshot.forEach((doc) => {
                    const userData = doc.data();
                    localStorage.setItem("id", userData.id);
                    localStorage.setItem("name", userData.name);
                    localStorage.setItem("email", userData.email);
                    localStorage.setItem("password", userData.password);
                    localStorage.setItem("photoURL", userData.URL);
                    localStorage.setItem("description", userData.description);
                    localStorage.setItem("FirebaseDocumentId", doc.id);
                  });
                } else {
                  this.errorMessage = "User data not found.";
                }
              }
            }
          } catch (error) {
            console.log(error);
          }

          if (data.role == "ROLE_GIANGVIEN") {
            this.$router.push("/teacher");
          } else if (data.role == "ROLE_SINHVIEN") {
            this.$router.push("/student");
          }
        }
      } catch (error) {
        if (error.response && error.response.status === 400) {
          this.errorMessage = "Tài khoản hoặc mật khẩu của bạn không đúng!!";
        } else {
          throw error;
        }
      }
    },
  },
  created() {
    if (localStorage.getItem("id")) this.$router.push("/student");
  },
  watch: {
    errorMessage: function (newErrorMessage, oldErrorMessage) {
      console.log(`Giá trị mới của user.errorMessage: ${newErrorMessage}`);
      console.log(`Giá trị cũ của user.errorMessage: ${oldErrorMessage}`);

      // Thực hiện các hành động khác tại đây nếu cần
    },
  },
};
</script>

