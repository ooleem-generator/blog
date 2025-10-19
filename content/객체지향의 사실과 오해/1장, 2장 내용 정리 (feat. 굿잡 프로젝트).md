---
tags:
  - 객체지향
---

객체지향의 사실과 오해 1장, 2장
- 1장 : 협력하는 객체들의 공동체
- 2장 : 이상한 나라의 객체

이 부분의 내용을 나만무 최종 프로젝트인 "올인원 커리어 플랫폼 - 굿잡"의 AI 모의면접 기능 개발 과정과 연관지어서 이해해보기로 했다.
(겸사겸사 NestJS, 프레임워크에 대한 공부도 할 겸)

## NestJS의 객체

NestJS의 컴포넌트는 클래스로 정의된다.
책에서 나온 커피숍 예시에서 고객이 클라이언트라고 하면,
캐시어는 Controller, 바리스타는 Provider이다.
NestJS는 각각을 싱글턴 패턴으로 하나의 객체로 만들기 때문에, 커피숍 비유와 연결시켜서 이해하기 좋은 것 같다. 고객의 요청을 받는 역할을 부여받은 캐시어 1명(컨트롤러 객체 하나), 실제 비즈니스 로직을 처리하는 역할을 부여받은 바리스타 1명(프로바이더 객체 하나) 이런 식으로. 

# 01. 협력하는 객체들의 공동체

1장의 내용 중, 전지전능한 객체(god object)를 만들지 말라는 내용이 있다. AI 모의면접 진행 시 클라이언트의 요청을 처리하는 다음 코드를 보자.

```typescript
// interview.controller.ts
@Post('question')

async createQuestion(@Body() body: unknown, @Req() req: any): Promise<QuestionResult> {
	this.logger.log(`POST /ai/question bodyKeys=${Object.keys((body as any) || {}).join(',')}`);
	const parsed = CreateQuestionBodySchema.safeParse(body);
	if (!parsed.success) {
		this.logger.warn(
		`createQuestion 스키마 오류: ${JSON.stringify(parsed.error.flatten())}`,
		);
		throw new BadRequestException(parsed.error.flatten());
	}
	
	const userId = Number((req as any).user_idx ?? (req as any).user?.idx);
	
	if (!userId) throw new BadRequestException('unauthorized');
	const anyData = parsed.data as any;
	const sessionId: string | undefined = anyData.sessionId;
	const jobPostUrl: string | undefined = anyData.jobPostUrl;
	
	if ('resumeFileId' in parsed.data) {
		this.logger.log(`createQuestion: resumeFileId=${parsed.data.resumeFileId}`);
		const summary = await this.resumeFiles.getSummaryById(parsed.data.resumeFileId, userId);
		if (!summary || summary.length < 10) {
		throw new BadRequestException('요약이 비어있습니다. 먼저 요약을 등록하세요.');
	}
	
	// 세션-이력서 연결: 세션이 있다면 external_key로 이력서 파일 id 저장
	if (sessionId && parsed.data.resumeFileId) {
		await this.db.execute(
		`INSERT INTO interview_sessions (session_id, user_id, external_key)
		VALUES (?, ?, ?)
		ON DUPLICATE KEY UPDATE
		external_key = IFNULL(external_key, VALUES(external_key))`,
		[sessionId, userId, parsed.data.resumeFileId],
		);
	}
	
	const result = await this.ai.createQuestionWithJobPost(summary, { sessionId, jobPostUrl });
	// 질문 생성과 동시에 questions.text 업서트
	if (sessionId) {
		// 세션 보장 (외래키 제약 대비)
		await this.db.execute(
		`INSERT IGNORE INTO interview_sessions (session_id, user_id) VALUES (?, ?)`,
		[sessionId, userId],
		);
		await this.db.execute(
		`INSERT INTO questions (session_id, question_id, order_no, text)
		VALUES (?, ?, ?, ?)
		ON DUPLICATE KEY UPDATE text = VALUES(text)`,
		[sessionId, result.question.id, 0, result.question.text],
		);
		}
		return result;
	}
	
	this.logger.log(
	`createQuestion: resumeSummaryLen=${parsed.data.resumeSummary.length}, jobPostUrl=${jobPostUrl ? 'Y' : 'N'}`,
	);
	
	const result = await this.ai.createQuestionWithJobPost(parsed.data.resumeSummary, {
		sessionId,
		jobPostUrl,
	});
	// 질문 생성과 동시에 questions.text 업서트 (세션이 있을 때만)
	if (sessionId) {
		await this.db.execute(
		`INSERT IGNORE INTO interview_sessions (session_id, user_id) VALUES (?, ?)`,
		[sessionId, userId],
		);
		await this.db.execute(
		`INSERT INTO questions (session_id, question_id, order_no, text)
		VALUES (?, ?, ?, ?)
		ON DUPLICATE KEY UPDATE text = VALUES(text)`,
		[sessionId, result.question.id, 0, result.question.text],
		);
	}
	return result;
}

```

분명히 캐시어의 덕목은 고객의 요청을 잘 받아서 바리스타에게 잘 넘겨주는 것일 텐데, 많은 일들을 자체적으로 처리하고 있는 것을 볼 수 있다. 

이제 리팩토링이 잘 되어 있는, 결과 리포트 부분의 캐시어를 보도록 하자.

```typescript
// report.controller.ts
@Controller('report')

export class ReportController {
	constructor(private readonly svc: ReportService) {}
	// 1. InterviewReport (메인) - overall_score만

	@Get(':sessionId/overall')
	async getOverall(@Param('sessionId') sessionId: string) {
	const overallScore = await this.svc.getOverallScore(sessionId);
	return { success: true, data: { overallScore } };

}

```

깔끔하게 바리스타에게 요청을 넘기는 역할에 충실하고 있다.

고객의 주문을 받는 캐시어 역할을 하는 컨트롤러는 바리스타 역할을 하는 프로바이더에게 메세지를 보낸다. 프로바이더가 어떻게 역할을 수행하는지는 관여하지 않는다.
위 코드에서는, `getOverallScore`라는 바리스타에게 `sessionId`라는 메시지를 보낸다. 그러면 바리스타는 이 세션 아이디에 해당하는 종합 점수 데이터를 보내줄 것이다.

이제 프로바이더가 어떻게 역할을 수행하는지 보자.

```typescript
// report.service.ts
async getOverallScore(sessionId: string): Promise<number> {

// 먼저 저장된 리포트에서 overall_score 조회
	const saved = await this.getSavedReport(sessionId);
		if (saved) {
			return saved.overall_score;
		}
		
// 저장된 리포트가 없으면 overall_score만 계산
	return this.computeOverallScoreOnly(sessionId);
}


```

`getOverallScore`는 다시 `getSavedReport`라는 객체에게 요청을 보낸다. 여러 객체가 연속적으로 협력하는 것을 볼 수 있다.

# 02. 이상한 나라의 객체

## 상태와 행동

AI 모의면접의 요청을 받는 `AiService` 객체의 상태는 다음과 같이 변수로 정의된다.
(결과 리포트를 발행해주는 `ReportService` 객체는 따로 상태에 해당하는 부분이 없었다)

```typescript
@Injectable()
export class AiService {
	private client: OpenAI;
	private readonly logger = new Logger(AiService.name);
	// 세션별 채용공고 컨텍스트 캐시 (프로세스 메모리)
	private readonly sessionJobCtx = new Map<
		string,
		{
		url?: string;
		title?: string;
		company?: string;
		content?: string;
		summary?: string;
		ts: number;
		}
>	();
	
	...

```

그리고 행동에 해당하는 부분은 다음과 같이 메소드로 정의되며, 행동에 의해 자신의 상태 `sessionJobCtx`를 변화시키는 것을 볼 수 있다.
```typescript
// 외부에서 세션 채용공고 요약을 주입할 수 있도록 공개 메서드 제공
setJobContext(
	sessionId: string,
	ctx: {
		url?: string;
		title?: string;
		company?: string;
		content?: string;
		summary?: string;
	},
	) {
		if (!sessionId) return;
		if (!ctx?.summary && !ctx?.content) return;
		this.sessionJobCtx.set(sessionId, { ...ctx, ts: Date.now() });
	}
	
	// 세션 종료/취소 시 메모리 컨텍스트 제거
	clearSessionContext(sessionId: string) {
		if (!sessionId) return;
		this.sessionJobCtx.delete(sessionId);
}

```



## 그런데 식별자는?

이 내용을 정리하면서 내 코드를 아무리 뒤져도 식별자 역할을 하는 부분을 찾을 수가 없었다.
알고 보니, NestJS에서는 자체적으로 프레임워크 내부에서 식별자를 관리하고 있었다.

(이하 ChatGPT의 설명)
> NestJS가 애플리케이션 시작 시 두 클래스를 싱글턴으로 인스턴스화하고 DI 컨테이너에 등록해 두기 때문에, 그 식별자는 프레임워크 내부(주입 토큰, 메모리 참조 등)에서 관리됩니다. 코드 바깥에서 특별히 따로 지정하거나 확인할 필요가 없고, 그래서 소스에서는 보이지 않는 거죠.

그런데.. 알고 보니 모듈에서 프로바이더를 불러올 때, 따로 식별자 역할을 하는 DI 토큰이라는 걸 등록할 수 있었다. 사용자가 토큰을 임의로 지정해서 커스텀 토큰으로 등록하면, 식별자 역할을 할 수 있는 것으로 보인다.

내 프로젝트에도 예시가 있었다. 내가 지정한 토큰은 아니고, NestJS에서 자체적으로 제공하는 토큰(`APP_GUARD`)이다.

```typescript
//app.modules.ts
providers: [
		AppService,
		DatabaseService,
		{
			provide: APP_GUARD,
			useClass: SessionGuard,
		}, 
		GcsService, 
	],

```





# 마무리

객체지향도, 프레임워크도 이제 막 공부를 시작하다 보니 책에 있는 모든 내용을 연관짓기 쉽지 않다. 뒷 내용을 읽으면서 내가 잘못 이해한 부분이 있다면 계속 수정해야겠다.
그래도 책을 읽으면서 프레임워크 공부를 같이 하니까 학습 효율이 더 좋아지는 게 느껴진다.