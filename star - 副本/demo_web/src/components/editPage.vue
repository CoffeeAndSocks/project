<template>
<el-button type="primary" @click="close">退出</el-button>
  <h1>{{questionnaire.theme}}：【{{getState(questionnaire.state)}}】</h1>
  <h2 v-if="questionnaire.state===1">
    发布时间：{{questionnaire.releaseTime}} <br> 截止时间：{{ questionnaire.endTime}}
  </h2>
  ---------------------------------------------------------<br>
    <div v-for="question in questions" :key="question.number" >

      问题{{question.number }}. {{question.title}} -{{getQuestionType(question.type)}} -{{getNecessary(question.necessary)}}

        <el-button type="text"  class="delete-button" @click="deleteQuestion(question.questionId)" >删除</el-button>
      <el-button type="text"   @click="modOn(question)">修改</el-button>
      <div v-if="showMod&&beMod(question.questionId)">
        <form @submit.prevent="handleSubmit">
          <input v-model="title" placeholder="问题标题" required><br>
          <div>是否必填：
            <label>
              <input type="radio" v-model="necessary" value="1" > 必填
            </label>
            <label>
              <input type="radio" v-model="necessary" value="0"> 非必填
            </label><br>
          </div>
          <!-- 这里可以添加更多关于问题的输入项 -->
        </form>
        <el-button v-if="showMod" type="primary" @click="modQuestion(question.questionId)">确定</el-button>
      </div>
      <div v-if="isChoice(question)">
        <div v-for="selection in selectionsForAll.at(question.number-1)" :key="selection.position">
          选项{{getSelectionPosition(selection.position)}}.{{selection.content}}
          <el-button type="text" class="delete-button" @click="deleteSelection(selection.selectionId)">删除</el-button>
          <el-button type="text" @click="modSelectionOn(selection)">修改</el-button>
          <div v-if="showModSelection&&selection.selectionId===beModSelectionId">
            <input v-model="content" placeholder="选项标题" required><br>
            <el-button type="primary" @click="modSelection(selection.selectionId)">确定</el-button>
          </div>
        </div>
        <el-button type="text" @click="addSelection(question.questionId)">添加选项</el-button>
      </div>

    <br>
      ---------------------------------------------------------
    </div>

  <!-- 添加问题按钮 -->
  <el-button type="primary" @click="addOn">添加问题</el-button>
  <div v-if="showAdd">
  <form @submit.prevent="handleSubmit">
    <input v-model="title" placeholder="问题标题" required><br>
    <div>是否必填：
    <label>
      <input type="radio" v-model="necessary" value="1" > 必填
    </label>
    <label>
      <input type="radio" v-model="necessary" value="0"> 非必填
    </label><br>
    </div>
    <div>问题类型：
    <select v-model="type">
      <option v-for="(value, key) in types" :key="key" :value="key">{{ getQuestionType(value) }}</option>
    </select>
    </div>
    <!-- 这里可以添加更多关于问题的输入项 -->
  </form>
    <el-button v-if="showAdd" type="primary" @click="addQuestion">确定</el-button>


  </div>
</template>>

<script>
import axios from "axios";

export default {
  props: ["id", "creatorId","close"],
  data() {
    return {
      showMod:false,
      showAdd:false,
      beModQuestionId:null,
      questions:[],
      necessary:1,
      types:[0,1,2,3],
      type:0,
      questionnaire: [],
      title: "",
      selectionsForAll:[],
      content:"选项",
      showModSelection:false,
      beModSelectionId:null
    };
  },

  mounted() {
    this.fetchQuestionnaire();

  },
  methods: {
    async fetchQuestionnaire() {
      console.log(this.creatorId);
        const response = await axios.get(`http://localhost:8090/getById?id=${this.id}`);
        this.questionnaire = response.data.data;
      let releaseTime=new Date(this.questionnaire.releaseTime)
      let endTime=new Date(this.questionnaire.endTime)

      console.log(releaseTime)
      this.questionnaire.releaseTime=releaseTime;
      this.questionnaire.endTime=endTime
        console.log(response)
      this.questions=[];
      await axios.get('http://localhost:8090/ListQuestionInIt?id='+this.id)
          .then(response=>{
            if(response.data.code===200) {
              const list = response.data.data;
              list.forEach(question => {

                this.questions.push(question);
              })
            }
            else
            {
              this.questions=[];
            }
          })
      this.selectionsForAll=[];
      const list=this.questions
     list.forEach(question=>{
         axios.get('http://localhost:8090/ListSelectionInIt?questionId='+question.questionId)
             .then(response=>{
               this.selectionsForAll[question.number-1]=response.data.data;
               });
             })

    },
    getQuestionType(type) {
      // 根据实际情况定义问题类型的映射
      const types = {
        0: '单选题',
        1: '多选题',
        2: '打分题(1~10)',
        3:'填空题'
      };
      return types[type] || '未知类型';
    },
    getSelectionPosition(position) {
      const positions= {
        1: 'A',
        2: 'B',
        3: 'C',
        4:'D',
        5:'E',
        6:'F',
        7:'G',
        8:'H'

      };
      return positions[position];
    },
    getState(state)
    {
      const states= {
        0: '未发布',
        1: '已发布',
        2: '已截止',
      };
      return states[state];
    },
    getNecessary(necessary)
    {
      const states= {
        0: '选填',
        1: '必填',
      };
      return states[necessary];
    },
   async addOn(){
     this.showAdd=true;
   },
    async addQuestion() {
      // 确单证问卷ID有效且有新问题标题
      try {
        const questionData = {
          "title": this.title, // 问题标题
          "necessary": this.necessary, // 假设默认不是必填
          "type": this.type, // 假设默认问题类型
        };
        console.log(questionData);
        // 发送请求添加问题
        const response = await axios.post(`http://localhost:8090/saveQuestion?id=`+this.id,questionData);

        if (response.status === 200) {
          console.log("问题添加成功");
          this.title = "";
          this.type=0;
          this.necessary=0;
          this.showAdd=false;
            await this.fetchQuestionnaire(); // 刷新问卷信
        } else {
          console.error("添加问题失败", response.data);
        }
      } catch (error) {
        console.error("添加问题时出错", error);

      }

    },
    async deleteQuestion(questionId){
console.log(questionId);
      const response = await axios.post(`http://localhost:8090/deleteQuestion?questionId=`+questionId);
      console.log(response);
      await this.fetchQuestionnaire();
    },
    modOn(question)
    {
      this.showMod=true;
      this.beModQuestionId=question.questionId;
      this.necessary=question.necessary;
      this.title=question.title;
      this.type=question.type
    },
    beMod(questionId)
    {
      if(questionId === this.beModQuestionId)
        return 1;
      else
        return 0;
    },
    async modQuestion(questionId)
    {
        const questionData = {
          "title": this.title, // 问题标题
          "necessary": this.necessary, // 假设默认不是必填
          "type" :this.type
        };
        console.log(questionData);
        // 发送请求添加问题
        const response = await axios.post(`http://localhost:8090/modQuestion?questionId=`+questionId,questionData);


          console.log(response);
          this.title = "";
          this.necessary=0;
          this.showMod=0;
          await this.fetchQuestionnaire(); // 刷新问卷信

      console.log(questionId)
    },

    isChoice(question)
    {
     if(question.type===1||question.type===0)
      {
        return 1;
      }
     else
       return false;


    },
  async deleteSelection(selectionId){
      const  response=await axios.post('http://localhost:8090/deleteSelection?selectionId='+selectionId);
      console.log(response)
      await this.fetchQuestionnaire();
  },
  async  addSelection(questionId) {
    const response = await axios.post('http://localhost:8090/saveSelection?questionId=' + questionId, {"content": "选项"});
    console.log(response);
    await this.fetchQuestionnaire();
  },
    modSelectionOn(selection)
    {
      this.showModSelection=true;
      this.beModSelectionId=selection.selectionId;
   this.content=selection.content
    },
    async modSelection(selectionId)
    {
      const response=await axios.post('http://localhost:8090/modSelection?selectionId='+selectionId+'&content='+this.content);
      console.log(response);
      this.showModSelection=false;
      await this.fetchQuestionnaire();
    }
  }
}
</script>
<style>
@import '@/css/border.css';
@import '@/css/bordertext.css';
@import '@/css/noUseButton.css';
@import '@/css/usingButton.css';
@import '@/css/buttonHover.css';
.delete-button{
  color: red;
}
.recordMessage {
  left: 550px;
  top: -700px;
  width: 1400px;
  border-radius: 30px;
  background: linear-gradient(135deg, rgba(174, 235, 198, 1) 0%, rgba(111, 227, 158, 0.01) 100%);
  position: relative;
  display: flex;
}
.recordMessage img{
  margin-left: 50px;
}

.recordMessage a {
  padding: 6px 12px;
  font-size: 14px;
  color: #007bff;
  text-decoration: none;
  border-radius: 4px;
  transition: background-color 0.3s ease;
}

.recordMessage a:hover {
  background-color: #e9ecef;
}

</style>